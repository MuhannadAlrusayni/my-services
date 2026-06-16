# SearXNG + Firecrawl Deployment Design

**Date:** 2026-06-16
**Purpose:** Deploy SearXNG (web search) and Firecrawl (web scraping) for use by the hermes AI agent running on the cluster node.

---

## Architecture Overview

Two independent namespaces, each managed as a plain-manifest Flux kustomization — consistent with the existing frigate/vaultwarden pattern.

```
clusters/production/
├── searxng/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── secret-sealed.yaml
│   └── kustomization.yaml
└── firecrawl/
    ├── namespace.yaml
    ├── configmap.yaml
    ├── api-deployment.yaml
    ├── worker-deployment.yaml
    ├── nuq-worker-deployment.yaml
    ├── playwright-deployment.yaml
    ├── redis-deployment.yaml
    ├── redis-pvc.yaml
    ├── nuq-postgres-deployment.yaml
    ├── nuq-postgres-pvc.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── secret-sealed.yaml
    └── kustomization.yaml
```

Hermes accesses both services via NodePort on the node's IP. Traefik ingress with cert-manager TLS exposes both externally.

---

## SearXNG

### Image
`docker.io/searxng/searxng:latest`

### Deployment
- Single replica
- Stateless — no PVC required
- Default settings.yaml baked into image
- Secret mounted as env var: `SEARXNG_SECRET_KEY`

### Service
- Type: `NodePort`
- Container port: `8080`
- NodePort: **9889** (fixed)
- Also accessible in-cluster on port 8080

### Secret (sealed)
- Key: `SEARXNG_SECRET_KEY`
- Value: random hex string (generate with `openssl rand -hex 32`)
- Stored as a SealedSecret

### Ingress
- Host: `searxng.malrusayni.com`
- Backend port: `8080`
- TLS: cert-manager `letsencrypt-prod` cluster issuer

---

## Firecrawl

Full stack per the [official self-hosting docs](https://github.com/firecrawl/firecrawl/blob/main/SELF_HOST.md) and [Kubernetes example](https://github.com/firecrawl/firecrawl/tree/main/examples/kubernetes/cluster-install).

### Images
| Component | Image |
|-----------|-------|
| API server | `ghcr.io/firecrawl/firecrawl:latest` |
| Worker | `ghcr.io/firecrawl/firecrawl:latest` (different entrypoint) |
| NUQ Worker | `ghcr.io/firecrawl/firecrawl:latest` (different entrypoint) |
| Playwright service | `ghcr.io/firecrawl/playwright-service:latest` |
| Redis | `redis:alpine` |
| Postgres | `ghcr.io/firecrawl/nuq-postgres:latest` |

### Deployments
Six deployments in the `firecrawl` namespace:

| Name | Command | Port | Notes |
|------|---------|------|-------|
| `api` | `node dist/src/index.js` | 3002 | Main HTTP API; `FLY_PROCESS_GROUP=app` |
| `worker` | `node dist/src/services/queue-worker.js` | 3005 (liveness) | Queue worker; `FLY_PROCESS_GROUP=worker` |
| `nuq-worker` | `node dist/src/services/worker/nuq-worker.js` | 3006 (liveness) | NUQ worker; `FLY_PROCESS_GROUP=nuq-worker`; 1 replica |
| `playwright-service` | default | 3000 | Browser renderer; separate image |
| `redis` | `redis-server` | 6379 | With optional password from secret |
| `nuq-postgres` | default | 5432 | Custom Postgres image; backed by PVC |

### ConfigMap (`firecrawl-config`)
| Key | Value |
|-----|-------|
| `HOST` | `0.0.0.0` |
| `REDIS_URL` | `redis://redis:6379` |
| `REDIS_RATE_LIMIT_URL` | `redis://redis:6379` |
| `PLAYWRIGHT_MICROSERVICE_URL` | `http://playwright-service:3000` |
| `USE_DB_AUTHENTICATION` | `false` |
| `ENV` | `production` |
| `IS_KUBERNETES` | `true` |
| `NUQ_DATABASE_URL` | `postgresql://firecrawl:<pw>@nuq-postgres:5432/firecrawl` |
| `PORT` | `3002` |

### Secret (sealed) — `firecrawl-secret`
| Key | Notes |
|-----|-------|
| `FIRECRAWL_API_KEY` | Random hex; hermes passes as `Authorization: Bearer <key>` (optional on self-hosted but enables auth) |
| `BULL_AUTH_KEY` | Random hex; protects the queue admin UI at `/admin/<key>/queues` |
| `REDIS_PASSWORD` | Random hex; Redis auth (optional but recommended) |
| `POSTGRES_USER` | `firecrawl` |
| `POSTGRES_PASSWORD` | Random hex |
| `POSTGRES_DB` | `firecrawl` |

All values sealed with `kubeseal` before committing.

### Services
- `api` — `NodePort`, port 3002 → NodePort **3002** (for hermes)
- `playwright-service` — `ClusterIP`, port 3000 (internal only)
- `redis` — `ClusterIP`, port 6379 (internal only)
- `nuq-postgres` — `ClusterIP`, port 5432 (internal only)
- Worker and nuq-worker have no Services (they pull from queue, no inbound traffic)

### PVCs
| Name | Size | Mounts to |
|------|------|-----------|
| `redis-pvc` | 1Gi | `/data` in redis pod |
| `nuq-postgres-pvc` | 2Gi | `/var/lib/postgresql/data` in postgres pod |

### Ingress
- Host: `firecrawl.malrusayni.com`
- Backend: `api` service port `3002`
- TLS: cert-manager `letsencrypt-prod` cluster issuer

---

## Hermes Access

| Service   | Access URL (from node)  | NodePort |
|-----------|-------------------------|----------|
| SearXNG   | `http://<node-ip>:9889` | 9889     |
| Firecrawl | `http://<node-ip>:3002` | 3002     |

Firecrawl API requests: `Authorization: Bearer <FIRECRAWL_API_KEY>` header.
Queue admin UI: `http://<node-ip>:3002/admin/<BULL_AUTH_KEY>/queues`

**SearXNG + Firecrawl integration:** Firecrawl's `/search` API can use SearXNG as its backend by setting `SEARXNG_ENDPOINT=http://searxng.searxng.svc.cluster.local:8080` in the Firecrawl ConfigMap. This is optional but means hermes can use a single Firecrawl call for both search and scrape.

---

## Deployment Approach

- Plain Kubernetes manifests + Kustomization (no Helm)
- Secrets sealed with `kubeseal` before committing
- Flux picks up changes on merge to `main`
- Each service's namespace added to the top-level `clusters/production` kustomization
