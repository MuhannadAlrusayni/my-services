# Harbor Container Registry — Design Spec

**Date:** 2026-06-20
**Status:** approved

## Overview

Deploy Harbor as an internal container registry on the production Kubernetes cluster.
Single-user (me) initially, with CI/CD integration possible later.
Full stack including Trivy vulnerability scanner.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Deployment method | Flux HelmRelease (official Harbor Helm chart) | Battle-tested, less maintenance, GitOps-native |
| Domain | `harbor.malrusayni.com` | Clearer for the Web UI, follows existing convention |
| Image storage | PVC (persistentVolumeClaim), 300Gi | Simple, no external dependency, matches repo pattern |
| Vulnerability scanning | Trivy enabled | Scan on push, useful now and for future CI/CD |
| PostgreSQL | External CloudNativePG, single instance, 5Gi | Robust, matches existing sanad pattern; small DB for personal use |
| Redis | Bundled via Helm chart | Zero extra manifests, self-contained |
| Secrets | SealedSecrets (two: admin password + PG credentials) | Matches repo convention |

## Architecture

### Namespace

`harbor` — single namespace for all Harbor resources.

### Components

| Component | Managed by | Notes |
|-----------|-----------|-------|
| Harbor Core | Helm chart | Auth, API, configuration |
| Harbor Portal | Helm chart | Web UI at `harbor.malrusayni.com` |
| Registry | Helm chart | Docker Distribution V2 API |
| Job Service | Helm chart | Garbage collection, replication, scanning |
| Trivy | Helm chart | Vulnerability scanner |
| Redis | Helm chart (bundled) | Session store + job queue |
| PostgreSQL | CloudNativePG Cluster | External DB, 5Gi PVC, single instance |

### Data Flow

```
User/CI  →  harbor.malrusayni.com  →  Traefik Ingress (TLS)
                                          ├── /api/*  →  Harbor Core
                                          └── /v2/*   →  Registry
                                                   ↓
                                          Registry Storage (300Gi PVC)
Harbor Core  →  Redis (session/queue)
Harbor Core  →  PostgreSQL (metadata, users, projects)
Trivy        →  Trivy DB PVC (5Gi)
```

### Boot Order

1. CloudNativePG operator creates `harbor-pg` Cluster → populates `harbor-pg-secret`
2. HelmRelease installs Harbor (references `harbor-pg-secret` for DB connection)

## Helm Configuration

### Chart

- **Repository:** `https://helm.goharbor.io`
- **Chart:** `harbor`
- **Flux interval:** 10m (HelmRelease), 1h (HelmRepository)

### Key Values

```yaml
database:
  type: external
  external:
    host: harbor-pg-rw.harbor.svc
    port: 5432
    username: harbor
    # password from harbor-pg-secret

redis:
  type: internal

persistence:
  enabled: true
  persistentVolumeClaim:
    registry:
      size: 300Gi

trivy:
  enabled: true

expose:
  type: ingress
  tls:
    enabled: false           # Traefik handles TLS
  ingress:
    hosts:
      core: harbor.malrusayni.com
    controller: none         # Custom ingress manifest
```

## Manifests

All under `clusters/production/harbor/`:

| File | Kind | Purpose |
|------|------|---------|
| `namespace.yaml` | Namespace | `harbor` namespace |
| `pg-secret-sealed.yaml` | SealedSecret | PostgreSQL username/password for CloudNativePG bootstrap |
| `pg-cluster.yaml` | Cluster (CNPG) | CloudNativePG PostgreSQL instance, 5Gi, single replica |
| `admin-secret-sealed.yaml` | SealedSecret | Harbor admin password (`HARBOR_ADMIN_PASSWORD`) |
| `helm-repository.yaml` | HelmRepository | Points to `https://helm.goharbor.io` |
| `helm-release.yaml` | HelmRelease | Harbor Helm chart with values, secrets refs |
| `ingress.yaml` | Ingress | Traefik ingress with cert-manager TLS |
| `kustomization.yaml` | Kustomization | Aggregates all resources |

No PVC or Service manifests needed — Helm chart creates those.

## Storage

| PVC | Size | Managed by |
|-----|------|-----------|
| Registry images | 300Gi | Helm chart |
| PostgreSQL | 5Gi | CloudNativePG Cluster |
| Trivy DB | 5Gi (chart default) | Helm chart |
| Job Service | 1Gi (chart default) | Helm chart |
| **Total** | **~311Gi** | |

## Secrets

### harbor-admin-secret
- Type: Opaque
- Key: `HARBOR_ADMIN_PASSWORD`
- Sealed via `kubeseal`, used as `valuesFrom` in HelmRelease

### harbor-pg-secret
- Type: Opaque
- Keys: `username`, `password`
- Sealed via `kubeseal`
- Used by: CloudNativePG bootstrap (`spec.bootstrap.initdb.secret.name`), HelmRelease (`database.external.password`)

## Prerequisites

- CloudNativePG operator already installed on cluster
- Flux source-controller and helm-controller operational
- SealedSecrets controller running
- Traefik + cert-manager (already in use)
- Cluster has ≥ 311Gi available storage

## Not In Scope

- OIDC/LDAP auth integration (Harbor local auth for now)
- Image replication / proxy cache to other registries
- CI/CD webhook integration (deferred)
- High-availability (single PG instance, single Harbor replica)
- Image retention policies / garbage collection schedule (chart defaults)
