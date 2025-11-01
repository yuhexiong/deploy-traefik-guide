# Deploy Traefik Guide

Use Traefik as a reverse proxy to automatically route frontend `/api` requests to the correct backend service.  
This way, the frontend no longer needs to bind to a fixed backend IP or domain during build time (no need to reset `.env` or rebuild),  
greatly simplifying the deployment process and improving flexibility.

## Overview

* Tool: Traefik v3.0

## Architecture

| Function  | Description                                                          |
| --------- | -------------------------------------------------------------------- |
| `/`       | Frontend page                                                        |
| `/api`    | Backend API                                                          |
| Port 3000 | External entry point (frontend and API proxy)                        |
| Port 8000 | Backend-only entry point during development (for Swagger)            |
| Dashboard | Enabled with `--api.dashboard=true`, used for development monitoring |

## Preparation

1. **Unify API path**
   Ensure all backend API paths start with `/api`, so Traefik can correctly determine routing.

2. **Build images**

## Run

After modifying the settings in [`docker-compose.yaml`](./docker-compose.yaml), start the entire stack:

```bash
docker-compose up -d
```

## Configuration

### Traefik

Traefik handles reverse proxying and routing control:

* Enable Docker provider for automatic container detection
* Only proxy services explicitly set with `traefik.enable=true`
* Create two entrypoints:

  * `web_frontend` (port 3000) → frontend entry
  * `web_api` (port 8000) → API and Swagger entry

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

> In production, it is recommended to disable `--api.insecure` and protect the dashboard with Basic Auth or another authentication method.

### Frontend Service

The frontend service only accepts requests coming from `web_frontend` (port 3000).  
Use `PathPrefix(`/`)` to proxy all static assets, with a lower priority (1).

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

The backend service is accessible from both:

* `web_frontend` (when the frontend calls `/api`)
* `web_api` (for direct API testing or viewing Swagger)

Set `PathPrefix(/api)` to distinguish routes, with a higher priority (2).  
Handle CORS through middleware.

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

* Accessing `http://<server>:3000` → displays the frontend page
* Frontend calls to `/api/...` → automatically routed to the backend service
* Backend service available at `http://<server>:8000` for testing or viewing Swagger

The entire deployment requires no rebuilds and no `.env` modification.  
Simply start the containers to complete deployment.
