# CI/CD Pipeline Setup Guide

This guide covers the setup and configuration of a CI/CD pipeline using Argo Events, Argo Workflows, and ArgoCD.

## Architecture Overview

- **Argo Events**: Receives webhooks from Bitbucket and triggers workflows based on branch and event type
- **Argo Workflows**: Executes CI pipeline (clone repo, build Docker image with Kaniko, push to Harbor)
- **ArgoCD**: Manages deployments across multiple environments (dev, staging, production)

## Prerequisites

1. Kubernetes cluster (1.19+)
2. Argo Events installed
3. Argo Workflows installed
4. ArgoCD installed
5. Harbor registry (self-hosted with self-signed certificate)
6. Git SSH key for repository access
7. Docker credentials for Harbor registry

## Setup Steps

### 1. Create Required Namespaces

The namespaces are already defined in the manifests:
- `argo-events`: For Argo Events components
- `argo-workflows`: For Argo Workflows components
- `argocd`: For ArgoCD components

### 2. Create SSH Key Secret for Git Access

```bash
# Create SSH key pair (if not already exists)
ssh-keygen -t ed25519 -f ~/.ssh/bitbucket_key -N ""

# Create the secret in argo-events namespace
kubectl create secret generic git-ssh-key \
  --from-file=ssh-privatekey=~/.ssh/bitbucket_key \
  -n argo-workflows

# Also create in argo-events namespace for sensor to reference
kubectl create secret generic git-ssh-key \
  --from-file=ssh-privatekey=~/.ssh/bitbucket_key \
  -n argo-events
```

### 3. Create Harbor Registry Secret

Create a `.dockerconfigjson` file for Harbor authentication:

```bash
# Create base64 encoded credentials
BASE64_CREDS=$(echo -n "username:password" | base64)

# Create the secret with insecure registry (self-signed cert)
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=admin@example.com \
  -n argo-workflows

# Also create in argo-events namespace
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=admin@example.com \
  -n argo-events
```

### 4. Deploy Argo Events Components

```bash
# Apply RBAC first
kubectl apply -f argo-events/rbac.yaml

# Apply EventSource
kubectl apply -f argo-events/event-source.yaml

# Apply Sensor
kubectl apply -f argo-events/sensor.yaml
```

### 5. Deploy Argo Workflows Components

```bash
# Apply RBAC
kubectl apply -f argo-workflows/rbac.yaml

# Apply WorkflowTemplate
kubectl apply -f argo-workflows/workflow-template.yaml
```

### 6. Deploy ArgoCD Components

```bash
# Apply ArgoCD ApplicationSet
kubectl apply -f argocd/applicationset.yaml
```

### 7. Configure Webhook in Bitbucket

Get the EventSource service IP/DNS:
```bash
kubectl get svc bitbucket-webhook -n argo-events
```

In Bitbucket repository settings:
- Add webhook URL: `http://<service-ip-or-dns>:12000/bitbucket`
- Select events:
  - `Repository Push` (for dev branch)
  - `Pull Request - Merged` (for main branch)

## Important Configuration Notes

### SSH Key Setup
- The SSH key must be added to the Bitbucket user's account
- The key format used here is Ed25519 (recommended), but RSA also works
- The SSH command in the workflow disables StrictHostKeyChecking for easier setup
- In production, consider using a deploy key specific to the repository

### Harbor Registry Configuration
- Registry uses self-signed certificate: `--skip-tls-verify=true` flag in Kaniko
- Kaniko executor uses insecure registry: `--insecure-registry=harbor.example.com`
- Image pull secrets are configured for the workflow pods
- Ensure the Kaniko image exists in Harbor or use a public Kaniko image

### Branch-Based Logic

**Main Branch (Production)**:
- Trigger: Pull Request Merged event (`pr:merged`)
- Branch detection: `toRef.displayId == 'main'`
- Image tags: `main-<commit-sha>` and `main-latest`

**Dev Branch (Development)**:
- Trigger: Repository Push event (`repo:refs_changed`)
- Branch detection: `changes[0].ref.displayId == 'dev'`
- Image tags: `dev-<commit-sha>` and `dev-latest`

### Workflow Execution Flow

1. **Clone Step**: Clones the repository at the specified branch
   - Outputs the commit SHA for image tagging
   - Uses SSH key for authentication
   
2. **Build & Push Step**: Uses Kaniko to build and push to Harbor
   - Builds from Dockerfile in repository root
   - Pushes to: `harbor.example.com/library/<repo-name>`
   - Tags with both commit-based and "latest" tags

### ArgoCD Deployment

The ApplicationSet creates three applications:
- `repository-development` → k8s/dev directory on dev branch
- `repository-staging` → k8s/staging directory on staging branch
- `repository-production` → k8s/production directory on main branch

Ensure your repository has the following structure:
```
repository/
├── Dockerfile
├── k8s/
│   ├── dev/
│   ├── staging/
│   └── production/
```

## Troubleshooting

### SSH Clone Failures
```bash
# Check if SSH key secret exists
kubectl get secret git-ssh-key -n argo-workflows

# Check workflow logs
kubectl logs <workflow-pod-name> -n argo-workflows -c clone-repo
```

### Harbor Push Failures
```bash
# Verify Harbor secret
kubectl get secret harbor-secret -n argo-workflows --output=yaml

# Check if Kaniko can access Harbor
# Verify certificate is properly handled with --skip-tls-verify=true
```

### Webhook Not Triggering
```bash
# Check EventSource service
kubectl get svc bitbucket-webhook -n argo-events

# Check Sensor logs
kubectl logs <sensor-pod-name> -n argo-events

# Verify webhook in Bitbucket is configured correctly
```

### Workflow Not Starting
```bash
# Check if Sensor is receiving events
kubectl get events -n argo-events --sort-by='.lastTimestamp'

# Verify Sensor condition expressions
kubectl describe sensor bitbucket-sensor -n argo-events
```

## Customization

### Adding New Branches
Modify the Sensor manifest to add new triggers with different branch conditions.

### Changing Image Registry
Replace `harbor.example.com` with your registry in:
- sensor.yaml
- workflow-template.yaml

### Modifying Build Parameters
Update the Kaniko executor command in workflow-template.yaml:
- Change base image with `--base-image`
- Add build arguments with `--build-arg`
- Use multi-stage builds in your Dockerfile

## Security Considerations

1. **SSH Keys**: Store in Kubernetes secrets, rotate regularly
2. **Registry Credentials**: Use strong passwords and consider using robot accounts in Harbor
3. **Network**: Restrict webhook endpoint access using NetworkPolicies
4. **RBAC**: Review and limit service account permissions as needed
5. **TLS**: Replace `--skip-tls-verify=true` with proper certificate handling in production
6. **Audit**: Enable Kubernetes audit logging for compliance tracking
