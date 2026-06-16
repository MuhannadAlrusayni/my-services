# SearXNG + Firecrawl Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy SearXNG and Firecrawl to the home Kubernetes cluster as Flux-managed plain-manifest kustomizations, with NodePort access for the hermes AI agent and Traefik ingress for external access.

**Architecture:** Two independent namespaces (`searxng`, `firecrawl`) each containing plain Kubernetes manifests managed by Flux GitOps. Firecrawl is wired to use SearXNG as its search backend via `SEARXNG_ENDPOINT`. Secrets are sealed with kubeseal using `~/.kube/my-home-cluster-sealed.pem`.

**Tech Stack:** Kubernetes, Flux CD, Kustomize, kubeseal (SealedSecrets), Traefik ingress, cert-manager

---

## Conventions

- All files go under `clusters/production/<namespace>/`
- Validate each namespace with: `KUBECONFIG=~/.kube/my-home-cluster.yaml kustomize build clusters/production/<ns>/`
- Seal secrets: `kubeseal --cert ~/.kube/my-home-cluster-sealed.pem --format yaml < secret.yaml > secret-sealed.yaml`
- Plaintext `secret.yaml` files are **never committed** — only `secret-sealed.yaml`

---

## Task 1: SearXNG — namespace, configmap, deployment, service, ingress

**Files:**
- Create: `clusters/production/searxng/namespace.yaml`
- Create: `clusters/production/searxng/configmap.yaml`
- Create: `clusters/production/searxng/deployment.yaml`
- Create: `clusters/production/searxng/service.yaml`
- Create: `clusters/production/searxng/ingress.yaml`

- [ ] **Step 1: Create namespace**

```yaml
# clusters/production/searxng/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: searxng
```

- [ ] **Step 2: Create ConfigMap with settings.yml**

```yaml
# clusters/production/searxng/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: searxng-config
  namespace: searxng
data:
  settings.yml: |
    use_default_settings: true
    server:
      limiter: false
      image_proxy: true
    search:
      formats:
        - html
        - json
```

- [ ] **Step 3: Create Deployment**

```yaml
# clusters/production/searxng/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: searxng
  namespace: searxng
spec:
  replicas: 1
  selector:
    matchLabels:
      app: searxng
  template:
    metadata:
      labels:
        app: searxng
    spec:
      containers:
        - name: searxng
          image: docker.io/searxng/searxng:latest
          ports:
            - containerPort: 8080
          env:
            - name: SEARXNG_SECRET
              valueFrom:
                secretKeyRef:
                  name: searxng-secret
                  key: SEARXNG_SECRET
          volumeMounts:
            - name: config
              mountPath: /etc/searxng
            - name: cache
              mountPath: /var/cache/searxng
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: config
          configMap:
            name: searxng-config
        - name: cache
          emptyDir: {}
```

- [ ] **Step 4: Create Service**

```yaml
# clusters/production/searxng/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: searxng
  namespace: searxng
spec:
  selector:
    app: searxng
  type: NodePort
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30889
```

- [ ] **Step 5: Create Ingress**

```yaml
# clusters/production/searxng/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: searxng
  namespace: searxng
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
    - host: "searxng.malrusayni.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: searxng
                port:
                  number: 8080
  tls:
    - hosts:
        - "searxng.malrusayni.com"
      secretName: searxng-tls
```

---

## Task 2: SearXNG — seal secret

**Files:**
- Create (do not commit): `clusters/production/searxng/secret.yaml`
- Create: `clusters/production/searxng/secret-sealed.yaml`

- [ ] **Step 1: Generate secret value**

```bash
SECRET_KEY=$(openssl rand -hex 32)
echo "SEARXNG_SECRET=$SECRET_KEY"
```

- [ ] **Step 2: Write plaintext secret**

```yaml
# clusters/production/searxng/secret.yaml  (DO NOT COMMIT)
apiVersion: v1
kind: Secret
metadata:
  name: searxng-secret
  namespace: searxng
type: Opaque
stringData:
  SEARXNG_SECRET: "<value from step 1>"
```

- [ ] **Step 3: Seal the secret**

```bash
kubeseal --cert ~/.kube/my-home-cluster-sealed.pem --format yaml \
  < clusters/production/searxng/secret.yaml \
  > clusters/production/searxng/secret-sealed.yaml
rm clusters/production/searxng/secret.yaml
```

---

## Task 3: SearXNG — kustomization and commit

**Files:**
- Create: `clusters/production/searxng/kustomization.yaml`

- [ ] **Step 1: Create kustomization.yaml**

```yaml
# clusters/production/searxng/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: searxng
resources:
  - namespace.yaml
  - configmap.yaml
  - secret-sealed.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

- [ ] **Step 2: Validate with kustomize**

```bash
kustomize build clusters/production/searxng/
```

Expected: all 6 resources printed with no errors.

- [ ] **Step 3: Commit**

```bash
git add clusters/production/searxng/
git commit -m "feat: deploy SearXNG with NodePort 9889 and Traefik ingress"
```

---

## Task 4: Firecrawl — namespace, configmap

**Files:**
- Create: `clusters/production/firecrawl/namespace.yaml`
- Create: `clusters/production/firecrawl/configmap.yaml`

- [ ] **Step 1: Create namespace**

```yaml
# clusters/production/firecrawl/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: firecrawl
```

- [ ] **Step 2: Create ConfigMap**

```yaml
# clusters/production/firecrawl/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: firecrawl-config
  namespace: firecrawl
data:
  HOST: "0.0.0.0"
  PORT: "3002"
  REDIS_URL: "redis://redis:6379"
  REDIS_RATE_LIMIT_URL: "redis://redis:6379"
  PLAYWRIGHT_MICROSERVICE_URL: "http://playwright-service:3000"
  USE_DB_AUTHENTICATION: "false"
  ENV: "production"
  IS_KUBERNETES: "true"
  SEARXNG_ENDPOINT: "http://searxng.searxng.svc.cluster.local:8080"
```

Note: `NUQ_DATABASE_URL` is in the secret (not here) because it contains credentials.

---

## Task 5: Firecrawl — Redis

**Files:**
- Create: `clusters/production/firecrawl/redis-pvc.yaml`
- Create: `clusters/production/firecrawl/redis-deployment.yaml`

- [ ] **Step 1: Create Redis PVC**

```yaml
# clusters/production/firecrawl/redis-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: firecrawl
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- [ ] **Step 2: Create Redis Deployment + Service**

```yaml
# clusters/production/firecrawl/redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: firecrawl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:alpine
          command: ["/bin/sh", "-c"]
          args:
            - |
              if [ -n "$REDIS_PASSWORD" ]; then
                exec redis-server --bind 0.0.0.0 --requirepass "$REDIS_PASSWORD" --save 60 1
              else
                exec redis-server --bind 0.0.0.0 --save 60 1
              fi
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: REDIS_PASSWORD
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: data
              mountPath: /data
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: firecrawl
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
```

---

## Task 6: Firecrawl — Postgres

**Files:**
- Create: `clusters/production/firecrawl/nuq-postgres-pvc.yaml`
- Create: `clusters/production/firecrawl/nuq-postgres-deployment.yaml`

- [ ] **Step 1: Create Postgres PVC**

```yaml
# clusters/production/firecrawl/nuq-postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nuq-postgres-pvc
  namespace: firecrawl
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

- [ ] **Step 2: Create Postgres Deployment + Service**

```yaml
# clusters/production/firecrawl/nuq-postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuq-postgres
  namespace: firecrawl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nuq-postgres
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nuq-postgres
    spec:
      containers:
        - name: nuq-postgres
          image: ghcr.io/firecrawl/nuq-postgres:latest
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: POSTGRES_DB
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nuq-postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nuq-postgres
  namespace: firecrawl
spec:
  selector:
    app: nuq-postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## Task 7: Firecrawl — Playwright service

**Files:**
- Create: `clusters/production/firecrawl/playwright-deployment.yaml`

- [ ] **Step 1: Create Playwright Deployment + Service**

```yaml
# clusters/production/firecrawl/playwright-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: playwright-service
  namespace: firecrawl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: playwright-service
  template:
    metadata:
      labels:
        app: playwright-service
    spec:
      containers:
        - name: playwright-service
          image: ghcr.io/firecrawl/playwright-service:latest
          ports:
            - containerPort: 3000
          env:
            - name: PORT
              value: "3000"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: playwright-service
  namespace: firecrawl
spec:
  selector:
    app: playwright-service
  ports:
    - port: 3000
      targetPort: 3000
```

---

## Task 8: Firecrawl — API server deployment + service + ingress

**Files:**
- Create: `clusters/production/firecrawl/api-deployment.yaml`
- Create: `clusters/production/firecrawl/service.yaml`
- Create: `clusters/production/firecrawl/ingress.yaml`

- [ ] **Step 1: Create API Deployment**

```yaml
# clusters/production/firecrawl/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: firecrawl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      terminationGracePeriodSeconds: 180
      containers:
        - name: api
          image: ghcr.io/firecrawl/firecrawl:latest
          command: ["node"]
          args: ["--max-old-space-size=4096", "dist/src/index.js"]
          ports:
            - containerPort: 3002
          env:
            - name: FLY_PROCESS_GROUP
              value: "app"
            - name: NUQ_DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: NUQ_DATABASE_URL
          envFrom:
            - configMapRef:
                name: firecrawl-config
            - secretRef:
                name: firecrawl-secret
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /v0/health/liveness
              port: 3002
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /v0/health/readiness
              port: 3002
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
```

- [ ] **Step 2: Create NodePort Service**

```yaml
# clusters/production/firecrawl/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: firecrawl
spec:
  selector:
    app: api
  type: NodePort
  ports:
    - name: http
      port: 3002
      targetPort: 3002
      nodePort: 30302
```

- [ ] **Step 3: Create Ingress**

```yaml
# clusters/production/firecrawl/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: firecrawl
  namespace: firecrawl
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
    - host: "firecrawl.malrusayni.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 3002
  tls:
    - hosts:
        - "firecrawl.malrusayni.com"
      secretName: firecrawl-tls
```

---

## Task 9: Firecrawl — Worker and NUQ Worker deployments

**Files:**
- Create: `clusters/production/firecrawl/worker-deployment.yaml`
- Create: `clusters/production/firecrawl/nuq-worker-deployment.yaml`

- [ ] **Step 1: Create Worker Deployment**

```yaml
# clusters/production/firecrawl/worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  namespace: firecrawl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: worker
          image: ghcr.io/firecrawl/firecrawl:latest
          command: ["node"]
          args: ["--max-old-space-size=3072", "dist/src/services/queue-worker.js"]
          env:
            - name: FLY_PROCESS_GROUP
              value: "worker"
            - name: PORT
              value: "3005"
            - name: NUQ_DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: NUQ_DATABASE_URL
          envFrom:
            - configMapRef:
                name: firecrawl-config
            - secretRef:
                name: firecrawl-secret
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "3Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /liveness
              port: 3005
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 3
```

- [ ] **Step 2: Create NUQ Worker Deployment**

```yaml
# clusters/production/firecrawl/nuq-worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuq-worker
  namespace: firecrawl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nuq-worker
  template:
    metadata:
      labels:
        app: nuq-worker
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: nuq-worker
          image: ghcr.io/firecrawl/firecrawl:latest
          command: ["node"]
          args: ["--max-old-space-size=3072", "dist/src/services/worker/nuq-worker.js"]
          env:
            - name: FLY_PROCESS_GROUP
              value: "nuq-worker"
            - name: NUQ_WORKER_PORT
              value: "3006"
            - name: NUQ_DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: firecrawl-secret
                  key: NUQ_DATABASE_URL
          envFrom:
            - configMapRef:
                name: firecrawl-config
            - secretRef:
                name: firecrawl-secret
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "3Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3006
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 3
```

---

## Task 10: Firecrawl — seal secret

**Files:**
- Create (do not commit): `clusters/production/firecrawl/secret.yaml`
- Create: `clusters/production/firecrawl/secret-sealed.yaml`

- [ ] **Step 1: Generate secret values**

```bash
FIRECRAWL_API_KEY=$(openssl rand -hex 32)
BULL_AUTH_KEY=$(openssl rand -hex 16)
REDIS_PASSWORD=$(openssl rand -hex 16)
POSTGRES_PASSWORD=$(openssl rand -hex 16)
echo "FIRECRAWL_API_KEY=$FIRECRAWL_API_KEY"
echo "BULL_AUTH_KEY=$BULL_AUTH_KEY"
echo "REDIS_PASSWORD=$REDIS_PASSWORD"
echo "POSTGRES_PASSWORD=$POSTGRES_PASSWORD"
```

- [ ] **Step 2: Write plaintext secret**

```yaml
# clusters/production/firecrawl/secret.yaml  (DO NOT COMMIT)
apiVersion: v1
kind: Secret
metadata:
  name: firecrawl-secret
  namespace: firecrawl
type: Opaque
stringData:
  FIRECRAWL_API_KEY: "<FIRECRAWL_API_KEY from step 1>"
  BULL_AUTH_KEY: "<BULL_AUTH_KEY from step 1>"
  REDIS_PASSWORD: "<REDIS_PASSWORD from step 1>"
  POSTGRES_USER: "firecrawl"
  POSTGRES_PASSWORD: "<POSTGRES_PASSWORD from step 1>"
  POSTGRES_DB: "firecrawl"
  NUQ_DATABASE_URL: "postgresql://firecrawl:<POSTGRES_PASSWORD from step 1>@nuq-postgres:5432/firecrawl"
```

- [ ] **Step 3: Seal the secret**

```bash
kubeseal --cert ~/.kube/my-home-cluster-sealed.pem --format yaml \
  < clusters/production/firecrawl/secret.yaml \
  > clusters/production/firecrawl/secret-sealed.yaml
rm clusters/production/firecrawl/secret.yaml
```

---

## Task 11: Firecrawl — kustomization and commit

**Files:**
- Create: `clusters/production/firecrawl/kustomization.yaml`

- [ ] **Step 1: Create kustomization.yaml**

```yaml
# clusters/production/firecrawl/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: firecrawl
resources:
  - namespace.yaml
  - configmap.yaml
  - secret-sealed.yaml
  - redis-pvc.yaml
  - redis-deployment.yaml
  - nuq-postgres-pvc.yaml
  - nuq-postgres-deployment.yaml
  - playwright-deployment.yaml
  - api-deployment.yaml
  - worker-deployment.yaml
  - nuq-worker-deployment.yaml
  - service.yaml
  - ingress.yaml
```

- [ ] **Step 2: Validate with kustomize**

```bash
kustomize build clusters/production/firecrawl/
```

Expected: all 16 resources printed with no errors.

- [ ] **Step 3: Commit**

```bash
git add clusters/production/firecrawl/
git commit -m "feat: deploy Firecrawl full stack with NodePort 3002 and Traefik ingress"
```
