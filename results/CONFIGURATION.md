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
- Commit SHA: `event.body.pullRequest.properties.mergeCommit.id`  ← the actual merge commit, NOT `toRef.latestCommit` (which is the tip of the target branch *before* the merge)
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

## EventSource Type

The EventSource uses the native `bitbucketserver:` type, **not** the generic `webhook:` type.

Key implications:
- **HMAC-SHA256 validation**: Bitbucket Server signs each request body and sends `X-Hub-Signature: sha256=<hmac>`. The event source recomputes the HMAC and rejects non-matching requests. This requires `webhookSecret` (see below), **not** `authSecret` (which only validates `Authorization: Bearer` — a header Bitbucket Server does not send).
- **`bitbucketserverBaseURL`**: Must be set to your Bitbucket Server REST API base URL (e.g. `https://bitbucket.example.com/rest`) before deploying. Required by the event source type even when `accessToken` is absent (passive mode).
- **No auto-registration**: `accessToken` is intentionally omitted. The webhook URL must be configured manually in Bitbucket Server (Project/Repository → Settings → Webhooks). The URL to register is the external URL of the `bitbucket-webhook-eventsource-svc` service on port 12000, path `/bitbucket`.

## Secret Requirements

### bitbucket-webhook-token Secret
Used by the `bitbucketserver:` EventSource for **HMAC-SHA256** validation of incoming webhook requests.

Bitbucket Server computes `HMAC-SHA256(body, secret)` and sends the result in the
`X-Hub-Signature` header. Argo Events recomputes it and rejects mismatches.

The `token` value here must match the **Secret** field configured in Bitbucket Server's webhook settings.

```bash
kubectl create secret generic bitbucket-webhook-token \
  --from-literal=token=<your-webhook-secret> -n argo-events
```

Created in:
- `argo-events` namespace only

### git-ssh-key Secret
Contents:
```
ssh-privatekey: <base64-encoded-private-key>
```

Created in:
- `argo-workflows` namespace
- `argo-events` namespace

### harbor-secret Secret
Docker registry secret with type: `kubernetes.io/dockerconfigjson`

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
