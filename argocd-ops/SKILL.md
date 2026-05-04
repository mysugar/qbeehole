---
name: argocd-ops
description: Argo CD operations and troubleshooting. Use this skill when you need to force synchronization of applications, resolve sync delays, or manage Argo CD resources via kubectl.
---

# Argo CD Operations Skill

This skill provides procedural knowledge for managing Argo CD applications and optimizing the GitOps synchronization loop.

## Core Operations

### 1. Force Application Sync (CLI)
When an application is out of sync or synchronization is delayed, use this `kubectl patch` command to force an immediate reconciliation. This effectively resets the `targetRevision` to `HEAD`, triggering a fresh manifest generation.

```bash
kubectl patch app <app-name> -n argocd --type merge -p '{"spec":{"source":{"targetRevision":"HEAD"}}}'
```

### 2. Hard Refresh vs. Refresh
- **Refresh**: Compares the cached Git state with the cluster state. Fast, but may miss very recent Git commits if the cache isn't updated.
- **Hard Refresh**: Clears the Git cache and re-clones the repository. Use this when `Refresh` fails to see new changes.

## Troubleshooting Sync Delays

### 1. Identify the Bottleneck
Check the application controller logs for connectivity issues or DNS timeouts:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### 2. Common Issues
- **DNS Timeouts**: Often caused by CoreDNS issues or network instability.
- **Image Pull Errors**: Kubernetes cannot start the `repo-server` or `application-controller` because it cannot pull the required images.
- **Backoff Logic**: Argo CD enters an exponential backoff state after repeated failures. A controller restart or a manual patch (see above) can bypass this.

## Optimization: Webhooks
To achieve sub-second synchronization, configure a Git Webhook.
1. Expose the `/api/webhook` endpoint of `argocd-server`.
2. Configure the URL in GitHub/GitLab repository settings.
3. **Security**: Configure a Webhook Secret in Argo CD's `argocd-secret`:
   ```bash
   kubectl patch secret argocd-secret -n argocd -p '{"stringData": {"webhook.github.secret": "YOUR_SECRET"}}'
   ```
4. Add the same secret to the GitHub Webhook settings.
