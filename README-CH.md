# Deploy Traefik Guide

使用 Traefik 作為反向代理，將前端的 `/api` 請求自動導向正確的後端服務。  
這樣一來，前端在 build 時不再需要綁死後端 IP 或 domain（不需重設 `.env` 或重新 build），  
可大幅簡化部署流程並提高靈活性。

## Overview

* Tool：Traefik v3.0

## Architecture

| 功能        | 說明                               |
| --------- | -------------------------------- |
| `/`       | 前端頁面                             |
| `/api`    | 後端 API                           |
| Port 3000 | 對外入口（前端與 API 代理）                 |
| Port 8000 | 開發階段後端專用入口（可開 Swagger）           |
| Dashboard | `--api.dashboard=true` 啟用，用於開發監控 |


## Preparation

1. **統一 API 路徑**  
   請確保所有後端 API 的路徑都以 `/api` 開頭，以便 Traefik 能正確進行路由判斷。

2. **前端 API 設定調整**  
   原本前端的 `.env` 可能設定為 `http://<server-ip>/api` 或類似形式。
   現在只需使用 `/api` 即可，實際的伺服器位置（IP 或網域）會由反向代理處理，不需要寫入 `.env` 或打包進建置結果中。

## Run

修改 [`docker-compose.yaml`](./docker-compose.yaml) 中的設定後，啟動整個架構：

```bash
docker-compose up -d
```

## Configuration

### Traefik 

Traefik 負責反向代理與路由控制：

* 啟用 Docker provider，自動偵測容器
* 僅對明確設定了 `traefik.enable=true` 的服務進行代理
* 建立兩個 entrypoints

  * `web_frontend`（port 3000）→ 前端入口
  * `web_api`（port 8000）→ API 與 Swagger 入口

```bash
--providers.docker=true
--providers.docker.exposedbydefault=false

--entrypoints.web_frontend.address=:3000
--entrypoints.web_api.address=:8000

--api.dashboard=true
--api.insecure=true

--accesslog=true
--accesslog.fields.defaultmode=drop
--accesslog.fields.names.EntryPoint=keep
--accesslog.fields.names.RouterName=keep
--accesslog.fields.names.ServiceName=keep
--accesslog.fields.names.RequestPath=keep
--accesslog.fields.names.DownstreamStatus=keep
```

> 正式環境建議關閉 `--api.insecure`，並改以 Basic Auth 或其他方式保護 dashboard。


### Frontend Service

前端服務僅接受從 `web_frontend`（port 3000）進來的請求。  
使用 `PathPrefix(/)` 代理所有靜態資源，優先度設為較低（1），優先度為數字大者優先。

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.server-frontend.rule=PathPrefix(`/`)"
  - "traefik.http.routers.server-frontend.entrypoints=web_frontend"
  - "traefik.http.routers.server-frontend.service=server-frontend"
  - "traefik.http.services.server-frontend.loadbalancer.server.port=3000"
  - "traefik.http.routers.server-frontend.priority=1"
```


### Backend Service

後端服務同時開放給：

* `web_frontend`（前端打 `/api` 時導向後端） 
* `web_api`（可直接測試 API 或查看 Swagger）

設定 `PathPrefix(/api)` 以區分路由，優先度設為 2，優先度為數字大者優先。  
並透過 middleware 處理 CORS。

```yaml
labels:
  - "traefik.enable=true"

  - "traefik.http.routers.server-api.rule=PathPrefix(`/api`)"
  - "traefik.http.routers.server-api.entrypoints=web_frontend,web_api"
  - "traefik.http.routers.server-api.service=server-api"
  - "traefik.http.services.server-api.loadbalancer.server.port=8000"
  - "traefik.http.routers.server-api.priority=2"

  # CORS Middleware
  - "traefik.http.routers.server-api.middlewares=cors-header@docker"
  - "traefik.http.middlewares.cors-header.headers.accessControlAllowOriginList=*"
  - "traefik.http.middlewares.cors-header.headers.accessControlAllowMethods=GET,OPTIONS,PUT,POST,DELETE"
  - "traefik.http.middlewares.cors-header.headers.accessControlAllowHeaders=*"
  - "traefik.http.middlewares.cors-header.headers.accessControlAllowCredentials=true"
```


## Result

* 使用者訪問 `http://<server>:3000` → 顯示前端頁面
* 前端呼叫 `/api/...` → 自動導向後端服務
* 後端服務可透過 `http://<server>:8000` 直接測試或檢視 Swagger

整體部署過程不需重新 build、也不需調整 `.env`  
僅需啟動容器即可完成部署。
