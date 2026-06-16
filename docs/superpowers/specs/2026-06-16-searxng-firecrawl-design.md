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
    ├── deployment.yaml        # API server
    ├── worker-deployment.yaml # Playwright worker
    ├── redis-deployment.yaml
    ├── redis-pvc.yaml
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

### Images
| Component | Image |
|-----------|-------|
| API server | `ghcr.io/mendableai/firecrawl:latest` |
| Worker | `ghcr.io/mendableai/firecrawl:latest` (different entrypoint) |
| Redis | `redis:alpine` |

### Deployments
Three separate Deployments in the `firecrawl` namespace:

1. **firecrawl-api** — HTTP API server on port 3002
2. **firecrawl-worker** — Playwright scraping worker; connects to Redis for job queue
3. **redis** — Single replica, backed by a 1Gi PVC

### Service
- Type: `NodePort`
- Container port: `3002` (API server only)
- NodePort: **3002** (fixed)
- Redis and worker are internal only (ClusterIP)

### Secret (sealed)
- Key: `FIRECRAWL_API_KEY`
- Value: random hex string (generate with `openssl rand -hex 32`)
- Shared by both `firecrawl-api` and `firecrawl-worker`
- Hermes uses this key in `Authorization: Bearer <key>` headers

### Redis PVC
- Size: 1Gi
- Default storage class
- Provides queue persistence across Redis pod restarts

### Ingress
- Host: `firecrawl.malrusayni.com`
- Backend port: `3002`
- TLS: cert-manager `letsencrypt-prod` cluster issuer

---

## Hermes Access

| Service  | Access URL (from node)          | NodePort |
|----------|---------------------------------|----------|
| SearXNG  | `http://<node-ip>:9889`         | 9889     |
| Firecrawl| `http://<node-ip>:3002`         | 3002     |

Firecrawl API requests require header: `Authorization: Bearer <FIRECRAWL_API_KEY>`

---

## Deployment Approach

- Plain Kubernetes manifests + Kustomization (no Helm)
- Secrets sealed with `kubeseal` before committing
- Flux picks up changes on merge to `main`
- Each service added to the top-level `clusters/production` kustomization
