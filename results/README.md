# Argo CI/CD Pipeline - Complete Implementation

This directory contains a production-ready CI/CD pipeline implementation using **Argo Events**, **Argo Workflows**, and **ArgoCD** to automate builds and deployments triggered by Bitbucket webhooks.

## 📁 Directory Structure

```
results/
├── argo-events/              # Event handling and webhook processing
│   ├── event-source.yaml     # Webhook listener configuration
│   ├── sensor.yaml           # Event rules and workflow triggers
│   └── rbac.yaml             # Service account and permissions
│
├── argo-workflows/           # CI pipeline execution
│   ├── workflow-template.yaml # Reusable CI workflow template
│   └── rbac.yaml             # Workflow execution permissions
│
├── argocd/                   # CD and deployment management
│   └── applicationset.yaml   # Multi-environment deployment config
│
├── SETUP_GUIDE.md            # Complete setup and deployment instructions
├── CONFIGURATION.md          # Detailed configuration reference
└── README.md                 # This file
```

## 🎯 Pipeline Overview

### Trigger Events
- **Main Branch**: Pull Request Merged → Image tagged as `main-<commit>`
- **Dev Branch**: Repository Push → Image tagged as `dev-<commit>`

### Workflow Steps
1. **Clone**: Pull code from Bitbucket using SSH
2. **Build**: Build Docker image with Kaniko
3. **Push**: Push to Harbor registry with semantic tagging
4. **Deploy**: ArgoCD manages deployments to appropriate environments

## 🚀 Quick Start

### 1. Prerequisites
```bash
# Kubernetes cluster with:
- Argo Events installed
- Argo Workflows installed
- ArgoCD installed
- Harbor registry accessible
- Git SSH key available
```

### 2. Create Required Secrets
```bash
# SSH key for Git repository access
kubectl create secret generic git-ssh-key \
  --from-file=ssh-privatekey=~/.ssh/bitbucket_key \
  -n argo-workflows

# Docker credentials for Harbor registry
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=<password> \
  -n argo-workflows
```

### 3. Deploy Manifests
```bash
# Argo Events
kubectl apply -f argo-events/rbac.yaml
kubectl apply -f argo-events/event-source.yaml
kubectl apply -f argo-events/sensor.yaml

# Argo Workflows
kubectl apply -f argo-workflows/rbac.yaml
kubectl apply -f argo-workflows/workflow-template.yaml

# ArgoCD
kubectl apply -f argocd/applicationset.yaml
```

### 4. Configure Bitbucket Webhook
Get the EventSource endpoint:
```bash
kubectl get svc bitbucket-webhook -n argo-events
```

Add webhook in Bitbucket: `http://<service-ip>:12000/bitbucket`

## 📋 Key Features

### Branch-Based Automation
- **Main Branch**: Controlled deployments from merged PRs
- **Dev Branch**: Continuous deployments on every push
- Extensible for additional branches (staging, release, etc.)

### Self-Signed Certificate Handling
- Kaniko configured for Harbor with `--skip-tls-verify=true`
- SSH configured to ignore host key verification for easier setup
- Production: Replace with proper certificate management

### Multi-Environment Deployment
- Development: Auto-syncs every build
- Staging: Optional intermediate environment
- Production: From main branch only

### SSH Key Authentication
- Ed25519 keys recommended (or RSA as fallback)
- Secure key storage in Kubernetes secrets
- ClusterRole permissions properly scoped

### Harbor Registry Integration
- Local registry with self-signed certificates
- Image pull secrets configured
- Semantic versioning: `branch-commitsha` and `branch-latest`

## 🔧 Configuration

See **CONFIGURATION.md** for:
- Webhook payload mappings
- Workflow parameter details
- Secret requirements
- Environment-specific settings
- Customization examples

See **SETUP_GUIDE.md** for:
- Detailed setup instructions
- Troubleshooting guide
- Security best practices
- Monitoring and logging

## 📊 Workflow Execution Flow

```
Bitbucket Event
    ↓
Argo Events (EventSource receives webhook)
    ↓
Sensor (evaluates branch and event type)
    ↓
Trigger Conditions Check (main branch PR? dev branch push?)
    ↓
Argo Workflow Created (with commit/repo parameters)
    ↓
Step 1: Clone Repository (SSH authentication)
    ↓
Step 2: Build & Push Image (Kaniko → Harbor)
    ↓
Workflow Complete
    ↓
ArgoCD Detects New Image
    ↓
Deployment Updated (environment-specific)
```

## 🔐 Security Considerations

1. **Credentials Management**
   - SSH key: Stored in Kubernetes secret `git-ssh-key`
   - Registry credentials: Stored in `harbor-secret`
   - Both scoped to required namespaces

2. **Network Access**
   - EventSource service exposes webhook endpoint
   - Recommend NetworkPolicies to restrict access
   - SSH port 22 to Bitbucket

3. **RBAC**
   - Workflows: Minimal permissions for pod execution
   - Events: Permissions to create workflows only
   - Review and adjust as needed

4. **Certificate Handling**
   - Development: Self-signed certificates with `--skip-tls-verify`
   - Production: Implement proper certificate verification

## 🛠️ Customization

### Change Registry URL
Replace `harbor.example.com` in all manifests with your registry.

### Add Additional Branches
Modify `sensor.yaml` to add triggers for other branches with different conditions.

### Modify Build Process
Update `workflow-template.yaml`:
- Change Dockerfile path
- Add build arguments
- Include additional steps (linting, testing, scanning)

### Add Notifications
Extend workflows with Slack/email notifications on success/failure.

## 📝 Manifest Details

### event-source.yaml
Exposes HTTP endpoint that listens for Bitbucket webhook events on port 12000.

### sensor.yaml
Contains two triggers:
1. **main-pr-merged-trigger**: Responds to merged PRs targeting main branch
2. **dev-push-trigger**: Responds to pushes to dev branch

Each trigger includes inline workflow definitions with branch-specific configurations.

### workflow-template.yaml
Reusable workflow template with:
- `clone-repo`: Git clone using SSH
- `build-push-image`: Kaniko image build and push

### applicationset.yaml
Generates three ArgoCD Applications:
- `repository-development` (dev branch, k8s/dev)
- `repository-staging` (staging branch, k8s/staging)
- `repository-production` (main branch, k8s/production)

### rbac.yaml Files
Define ServiceAccounts and Roles with minimum required permissions for:
- Creating and managing workflows
- Accessing secrets
- Pod execution

## ❓ Troubleshooting

See **SETUP_GUIDE.md** for detailed troubleshooting of:
- SSH clone failures
- Harbor push failures
- Webhook not triggering
- Workflow not starting

Common checks:
```bash
# Verify EventSource
kubectl get eventsources -n argo-events

# Verify Sensor
kubectl get sensors -n argo-events

# Check workflow status
kubectl get workflows -n argo-workflows

# View workflow logs
argo logs <workflow-name> -n argo-workflows
```

## 📚 References

- [Argo Events Documentation](https://argoproj.github.io/argo-events/)
- [Argo Workflows Documentation](https://argoproj.github.io/argo-workflows/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kaniko Documentation](https://github.com/GoogleContainerTools/kaniko)
- [Harbor Documentation](https://goharbor.io/docs/)

## 📄 License

This pipeline implementation is provided as-is for use with Argo tooling.
