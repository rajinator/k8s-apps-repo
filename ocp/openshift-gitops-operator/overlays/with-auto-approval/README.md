# OpenShift GitOps Operator - Auto-Approval Overlay

This overlay uses a **CronJob** for automatic InstallPlan approval, suitable for non-production environments or when you trust operator upgrades.

## Differences from Base

| Resource | Base | This Overlay |
|----------|------|--------------|
| InstallPlan Approver | One-time Job | CronJob (every 5 min) |
| Approval Strategy | Manual after bootstrap | Automatic ongoing |
| Use Case | Production clusters | Dev/homelab clusters |

## How It Works

### CronJob Configuration
```yaml
schedule: "*/5 * * * *"  # Runs every 5 minutes
concurrencyPolicy: Forbid
ttlSecondsAfterFinished: 600  # Cleanup completed jobs
```

### Operator Update Flow
1. Update `startingCSV` in Git
2. Push to repository
3. ArgoCD syncs new Subscription
4. OLM creates pending InstallPlan
5. Within 5 minutes, CronJob approves it
6. Operator upgrades automatically

## Deployment

### For Self-Managed GitOps

Update the Application to use this overlay:

```yaml
# In k8s-gitops-repo/apps/components/openshift-gitops-self-managed/
spec:
  source:
    path: ocp/openshift-gitops-operator/overlays/with-auto-approval
```

### Direct Deployment

```bash
oc apply -k ocp/openshift-gitops-operator/overlays/with-auto-approval/
```

## Monitoring

```bash
# Check CronJob status
oc get cronjob installplan-approver -n openshift-gitops-operator

# View recent jobs
oc get jobs -n openshift-gitops-operator -l app=installplan-approver

# Check logs
oc logs -l app=installplan-approver -n openshift-gitops-operator --tail=50
```

## Switching Back to Manual

To switch back to manual approval:

1. Change Application source path to `overlays/self-managed`
2. Delete the CronJob:
   ```bash
   oc delete cronjob installplan-approver -n openshift-gitops-operator
   ```
3. Approve InstallPlans manually going forward

## Security Considerations

⚠️ **Auto-approval means:**
- Operator upgrades happen automatically within 5 minutes
- No human gate for version changes
- Suitable for homelab/dev, use caution in production

✅ **Mitigation strategies:**
- Pin `startingCSV` versions carefully
- Test upgrades in dev first
- Monitor operator health after upgrades
- Use Git branch protection for operator manifests

