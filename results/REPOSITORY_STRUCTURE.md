# Expected Repository Structure

This document describes the expected structure of your application repository that will be cloned and built by the CI/CD pipeline.

## Bitbucket Repository Layout

```
my-application/
├── Dockerfile                    # Required: Docker image definition
├── src/                          # Application source code
│   ├── main.py
│   ├── app.py
│   └── ...
├── tests/                        # Test files
│   └── test_app.py
├── requirements.txt              # Python dependencies (or equivalent)
├── README.md                     # Project documentation
└── k8s/                          # Kubernetes manifests for deployments
    ├── dev/                      # Development environment
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yaml
    │   └── kustomization.yaml
    ├── staging/                  # Staging environment
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── production/               # Production environment
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        └── kustomization.yaml
```

## Dockerfile Example

```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY src/ .

EXPOSE 8000
CMD ["python", "main.py"]
```

## Kubernetes Manifests Structure

### Development Deployment (k8s/dev/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-application
      env: dev
  template:
    metadata:
      labels:
        app: my-application
        env: dev
    spec:
      containers:
      - name: app
        image: harbor.example.com/library/my-application:dev-latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        env:
        - name: ENVIRONMENT
          value: "development"
      imagePullSecrets:
      - name: harbor-secret
```

### Production Deployment (k8s/production/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-application
      env: production
  template:
    metadata:
      labels:
        app: my-application
        env: production
    spec:
      containers:
      - name: app
        image: harbor.example.com/library/my-application:main-latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
      imagePullSecrets:
      - name: harbor-secret
```

### Service Example (k8s/dev/service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-application
  namespace: development
spec:
  type: ClusterIP
  selector:
    app: my-application
    env: dev
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
```

### ConfigMap Example (k8s/dev/configmap.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-application-config
  namespace: development
data:
  LOG_LEVEL: "INFO"
  DATABASE_POOL_SIZE: "10"
```

## Kustomization Files

### k8s/dev/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

namePrefix: dev-

commonLabels:
  env: development
  app: my-application
```

### k8s/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - ingress.yaml

commonLabels:
  env: production
  app: my-application
```

## CI/CD Workflow Integration

### How the Pipeline Uses This Structure

1. **Clone Step**
   - Clones the repository at the triggered branch
   - Default branch: `main` or `dev`

2. **Build Step**
   - Reads `Dockerfile` from repository root
   - Builds image with Kaniko
   - Context: repository root

3. **Push Step**
   - Pushes to: `harbor.example.com/library/my-application:<branch>-<commit>`
   - Also tags as: `harbor.example.com/library/my-application:<branch>-latest`

4. **Deploy Step (ArgoCD)**
   - Reads manifests from `k8s/<environment>/<branch>` directory
   - Applies Kustomization if `kustomization.yaml` exists
   - Updates image reference in deployment
   - Syncs to cluster

## Image Update Flow

### Development Branch
```
Push to dev branch
    ↓
Workflow clones dev branch
    ↓
Build: my-application:dev-abc123
    ↓
Push: harbor.example.com/library/my-application:dev-abc123
       harbor.example.com/library/my-application:dev-latest
    ↓
ArgoCD detects new image tag
    ↓
Updates deployment in k8s/dev/ with new image
    ↓
Deployment updates in development namespace
```

### Main Branch
```
PR merged to main branch
    ↓
Workflow clones main branch
    ↓
Build: my-application:main-def456
    ↓
Push: harbor.example.com/library/my-application:main-def456
       harbor.example.com/library/my-application:main-latest
    ↓
ArgoCD detects new image tag
    ↓
Updates deployment in k8s/production/ with new image
    ↓
Deployment updates in production namespace
```

## Important Notes

1. **Dockerfile Location**
   - Must be at repository root
   - Pipeline doesn't support custom paths currently
   - Modify sensor.yaml if you need different structure

2. **Kubernetes Manifests**
   - Must be in `k8s/<environment>/` directories
   - ArgoCD reads from these paths
   - Environment names: `dev`, `staging`, `production`

3. **Image Tag Convention**
   - Development: `dev-<commit-sha>` (short or full)
   - Production: `main-<commit-sha>`
   - Always includes `<branch>-latest` tag
   - Update deployments to use `:latest` tags for automatic updates

4. **Secret Management**
   - Create `harbor-secret` in each namespace for image pulls
   - Keep SSH keys secure in repository settings
   - Don't commit secrets to repository

5. **Multi-Branch Support**
   - Current setup: main (production) and dev (development)
   - Add staging branch:
     - Create `k8s/staging/` directory
     - Add trigger in `sensor.yaml`
     - Create separate ApplicationSet generator entry

## Example: Adding Staging Branch

### Step 1: Create k8s/staging/ directory
```bash
mkdir -p k8s/staging
# Copy files from k8s/dev/ and modify as needed
```

### Step 2: Update sensor.yaml
Add a new trigger for staging branch:
```yaml
- template:
    name: staging-push-trigger
    conditions: "bitbucket-event"
    when:
      all:
        - expr: "event.body.eventKey == 'repo:refs_changed' && event.body.changes[0].ref.displayId == 'staging'"
    # ... rest of trigger configuration
```

### Step 3: Update applicationset.yaml
Add staging to generators:
```yaml
generators:
  - list:
      elements:
        - env: development
          branch: dev
          repo: my-application
        - env: staging
          branch: staging
          repo: my-application
        - env: production
          branch: main
          repo: my-application
```

## Dockerfile Best Practices

1. **Multi-stage builds** (recommended)
   - Reduces final image size
   - Separates build dependencies from runtime

2. **Use specific base image tags**
   - ❌ Avoid: `FROM python:latest`
   - ✅ Use: `FROM python:3.11-slim`

3. **Layer caching optimization**
   - Order: dependencies → code
   - Changes less frequently first

4. **Security scanning**
   - Use Harbor's built-in vulnerability scanning
   - Address CVEs before production deploy

## Troubleshooting Repository Structure Issues

### Error: "Dockerfile not found"
- Ensure `Dockerfile` is at repository root
- Check file name case sensitivity

### ArgoCD not syncing
- Verify `k8s/<environment>/` directories exist
- Check `kustomization.yaml` syntax
- Ensure namespace matches (defined in kustomization)

### Wrong image being deployed
- Verify deployment manifest uses `:latest` tag or specific tag
- Check Harbor for correct image tags
- Confirm ArgoCD image update automation is enabled
