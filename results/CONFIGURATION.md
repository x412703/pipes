# Configuration Reference

## Webhook Payload Mapping

### PR Merged Event (Main Branch)

Bitbucket sends:
```json
{
  "eventKey": "pr:merged",
  "pullRequest": {
    "toRef": {
      "displayId": "main",
      "latestCommit": "7e48f426f0a6e47c5b5e862c31be6ca965f82c9c",
      "repository": {
        "slug": "repository"
      }
    }
  }
}
```

Sensor extracts and passes to Workflow:
- Commit SHA: `event.body.pullRequest.toRef.latestCommit`
- Repository name: `event.body.pullRequest.toRef.repository.slug`
- Branch: `main` (hardcoded in trigger)

### Repo Push Event (Dev Branch)

Bitbucket sends:
```json
{
  "eventKey": "repo:refs_changed",
  "changes": [{
    "ref": {
      "displayId": "dev"
    },
    "toHash": "178864a7d521b6f5e720b386b2c2b0ef8563e0dc"
  }],
  "repository": {
    "slug": "repository"
  }
}
```

Sensor extracts and passes to Workflow:
- Commit SHA: `event.body.changes[0].toHash`
- Repository name: `event.body.repository.slug`
- Branch: `dev` (hardcoded in trigger)

## Workflow Parameters

Each workflow receives:
- `commit-sha`: The commit hash to build
- `repo-name`: The repository name (used for image naming)
- `branch`: The branch name (main or dev)

## Image Naming Convention

```
harbor.example.com/library/<repo-name>:<branch>-<commit-sha>
harbor.example.com/library/<repo-name>:<branch>-latest
```

Examples:
- `harbor.example.com/library/repository:main-7e48f426f0a6e47c5b5e862c31be6ca965f82c9c`
- `harbor.example.com/library/repository:main-latest`
- `harbor.example.com/library/repository:dev-178864a7d521b6f5e720b386b2c2b0ef8563e0dc`
- `harbor.example.com/library/repository:dev-latest`

## Secret Requirements

### git-ssh-key Secret
Contents:
```
ssh-privatekey: <base64-encoded-private-key>
```

Created in:
- `argo-workflows` namespace
- `argo-events` namespace

### harbor-secret Secret
Docker registry secret with type: `kubernetes.io/dockercfg`

Contents:
```
.dockerconfigjson: <base64-encoded-docker-config>
```

Or use this command:
```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=<password> \
  --docker-email=admin@example.com
```

Created in:
- `argo-workflows` namespace
- `argo-events` namespace (for Sensor to reference)

## Kubernetes Context Requirements

The clusters/namespaces must have:
1. Argo Events EventBus named `default` (created during Argo Events installation)
2. Argo Workflows controller running
3. PersistentVolume support for artifacts (if used)

## Environment-Specific Configuration

### Production (Main Branch)
- Triggers on merged pull requests
- More conservative with deployments
- Manual approval steps can be added in ArgoCD

### Development (Dev Branch)
- Triggers on every push
- Auto-syncs in ArgoCD
- Fast feedback loop

### Staging (Optional)
- Can be added as separate trigger
- Intermediate testing environment
- Manual promotion to production

## Monitoring and Logging

### Argo Events
- Check EventSource: `kubectl get eventsources -n argo-events`
- Check Sensor: `kubectl get sensors -n argo-events`
- View logs: `kubectl logs -f <pod-name> -n argo-events`

### Argo Workflows
- Check workflows: `kubectl get workflows -n argo-workflows`
- View workflow: `argo workflows get <workflow-name> -n argo-workflows`
- View logs: `argo logs <workflow-name> -n argo-workflows`

### ArgoCD
- Check applications: `kubectl get applications -n argocd`
- View sync status: `argocd app list`
- Check logs: `kubectl logs -f <pod-name> -n argocd`

## Customization Examples

### Adding Slack Notification on Build Failure
Add a new template in workflow-template.yaml:
```yaml
- name: notify-slack
  container:
    image: curlimages/curl:latest
    command: [sh, -c]
    args:
      - |
        curl -X POST $SLACK_WEBHOOK_URL \
          -H 'Content-Type: application/json' \
          -d '{"text":"Build failed for {{workflow.parameters.repo-name}}"}'
    env:
      - name: SLACK_WEBHOOK_URL
        valueFrom:
          secretKeyRef:
            name: slack-webhook
            key: url
```

### Using a Different Docker Registry
Replace all occurrences of `harbor.example.com` with your registry URL and update credentials accordingly.

### Adding Code Quality Checks
Add additional steps in the ci-pipeline template before pushing:
```yaml
- - name: code-quality-check
    template: quality-check
- - name: build-push-image
    template: build-push-image
```

### Supporting Additional Branches
Add more triggers to the Sensor for other branches (e.g., staging, release branches).
