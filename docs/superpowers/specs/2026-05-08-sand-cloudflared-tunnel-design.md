# Design: sand-cloudflared-tunnel Deployment

**Date:** 2026-05-08
**Status:** Approved

## Overview

Deploy a new Cloudflare tunnel for the `sand` service in the `faris` cluster, following the exact same pattern as the existing `clusters/production/faris/cloudflared-tunnel/` deployment.

## Directory Structure

```
clusters/production/faris/sand-cloudflared-tunnel/
├── namespace.yaml
├── deployment.yaml
├── cloudflared-token-secret-sealed.yaml   # committed
├── cloudflared-token-secret.yaml          # local only, gitignored
└── kustomization.yaml
```

## Files

### namespace.yaml

Kubernetes Namespace named `faris-sand-cloudflared-tunnel`.

### deployment.yaml

Identical to the template deployment except namespace is `faris-sand-cloudflared-tunnel`. Runs `cloudflare/cloudflared:latest` with `--no-autoupdate` and reads `TUNNEL_TOKEN` from the `cloudflared-token` SealedSecret.

Resources: 50m/64Mi requests, 200m/128Mi limits.

### cloudflared-token-secret.yaml

Raw Kubernetes Secret with the tunnel token — used to pipe into `kubeseal`. Committed to git, matching the pattern of the existing template (the template's raw secret is also tracked).

### cloudflared-token-secret-sealed.yaml

SealedSecret produced by kubeseal from the raw secret above. Namespace-scoped to `faris-sand-cloudflared-tunnel`.

### kustomization.yaml

Lists three resources: `namespace.yaml`, `cloudflared-token-secret-sealed.yaml`, `deployment.yaml`.

## Flux Discovery

No parent kustomization changes are needed. The Flux `Kustomization` CRD in `gotk-sync.yaml` points to `./clusters/production` and recursively discovers all `kustomization.yaml` files in subdirectories, so the new directory will be picked up automatically on the next sync.

## Secret Handling

The provided tunnel token is sealed with `kubeseal` targeting the `faris-sand-cloudflared-tunnel` namespace. Both the raw secret file (`cloudflared-token-secret.yaml`) and the sealed secret (`cloudflared-token-secret-sealed.yaml`) are committed to git, matching the pattern of the existing template.
