# GitLab Runner Operator - Auto Approval Overlay

This overlay deploys the GitLab Runner operator with **automatic InstallPlan approval** via CronJob.

## What's Included

- Base operator resources (Namespace, OperatorGroup, Subscription)
- Namespace patches (GitOps labels, sync-wave)
- **CronJob** for automatic InstallPlan approval (runs every 5 minutes)

## Use Cases

✅ **Homelab/Development**: Automatic operator upgrades without manual intervention

❌ **Production**: Use `overlays/gitops` instead (manual approval for control)

## Deployment

### Via ArgoCD (GitOps)

Use the `gitlab-runner-with-auto-approval` component in your cluster config:

```yaml
# ocp-gitops-repo/apps/dev-cluster/kustomization.yaml
components:
  - ../components/gitlab-runner-with-auto-approval
```

### Manual

```bash
# Deploy everything including CronJob
oc apply -k overlays/with-auto-approval/
```

## CronJob Behavior

- **Schedule**: Every 5 minutes
- **Action**: Checks for pending InstallPlans and approves them
- **Concurrency**: Forbid (prevents overlapping runs)
- **History**: Keeps last 1 successful and 1 failed job

## Compared to Other Overlays

| Overlay | InstallPlan Approval | Use Case |
|---------|---------------------|----------|
| `overlays/gitops` | Manual | Production (control) |
| `overlays/with-auto-approval` | CronJob | Homelab (automated) |

## Notes

- CronJob requires RBAC to patch InstallPlans (included)
- Operator upgrades happen automatically when OLM releases updates
- Check CronJob logs: `oc logs -l app=installplan-approver -n gitlab-runner-operator`

