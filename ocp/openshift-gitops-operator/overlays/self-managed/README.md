# Self-Managed OpenShift GitOps Operator

This overlay creates an ArgoCD Application that makes the OpenShift GitOps operator manage itself.

## Concept

"GitOps manages GitOps" - Once the operator is initially installed manually, this Application takes over management, making the operator fully declarative and managed via Git.

## Deployment Order

### 1. Initial Manual Install (Bootstrap)

```bash
# First-time installation
oc apply -k ../../base/
```

Wait for operator to be ready:
```bash
oc get csv -n openshift-gitops-operator
oc get pods -n openshift-gitops
```

### 2. Apply Custom Configuration

```bash
oc apply -k ../custom/
```

### 3. Enable Self-Management

```bash
# Deploy the self-management Application
oc apply -f self-managed-app.yaml
```

From this point forward, ArgoCD will manage the operator and sync any changes from Git.

## How It Works

1. The `self-managed-app.yaml` creates an ArgoCD Application
2. The Application points to `ocp/openshift-gitops-operator/base/`
3. **Excludes InstallPlan job** - Job only runs during bootstrap, not via GitOps
4. ArgoCD now syncs operator configuration from Git
5. Changes to Subscription, RBAC, etc. are automatically applied

### Why Exclude the InstallPlan Job?

```yaml
directory:
  exclude: 'installplan-approver-job.yaml'
```

The InstallPlan approver job should only run **once** during initial bootstrap. If managed by ArgoCD:

1. Job completes successfully
2. Job stays in cluster (no TTL)
3. If job had `ttlSecondsAfterFinished`, Kubernetes would delete it
4. ArgoCD would detect it's missing and recreate it
5. **Infinite loop** ❌

By excluding it, the job runs once during bootstrap and is never touched by GitOps.

## Features

- **Automated Self-Healing**: ArgoCD will revert manual changes
- **Git as Source of Truth**: All operator config in Git
- **Declarative Upgrades**: Update `startingCSV` in Git to upgrade
- **Ignore OLM Status**: Prevents sync issues with OLM-managed status fields

## Important Notes

### Prune is Disabled
```yaml
automated:
  prune: false  # Don't delete operator resources automatically
```
This prevents ArgoCD from accidentally deleting the operator that manages it.

### Server-Side Apply
```yaml
syncOptions:
  - ServerSideApply=true
```
Better conflict resolution when OLM and ArgoCD both manage resources.

### Ignore Differences
```yaml
ignoreDifferences:
  - group: operators.coreos.com
    kind: Subscription
    jsonPointers:
      - /status
```
OLM manages status fields; ArgoCD should ignore these.

## Updating the Operator

### Option 1: Manual Approval (Current Setup)

```bash
# 1. Update startingCSV in base/subscription.yaml
vim ../../base/subscription.yaml
# Change: startingCSV: openshift-gitops-operator.v1.18.1
# To:     startingCSV: openshift-gitops-operator.v1.19.0

# 2. Commit and push to Git
git add ../../base/subscription.yaml
git commit -m "Upgrade GitOps operator to v1.19.0"
git push

# 3. ArgoCD syncs, OLM creates new pending InstallPlan

# 4. Manually approve the InstallPlan
oc get installplan -n openshift-gitops-operator
oc patch installplan <plan-name> -n openshift-gitops-operator \
  --type merge -p '{"spec":{"approved":true}}'
```

### Option 2: Automatic Approval (Recommended for Homelab)

Use the `with-auto-approval` overlay instead:

```bash
# Switch to auto-approval overlay
# Update self-managed Application source path:
path: ocp/openshift-gitops-operator/overlays/with-auto-approval

# CronJob will auto-approve within 5 minutes of any pending InstallPlan
```

See `../with-auto-approval/README.md` for details.

## Verifying Self-Management

```bash
# Check the Application status
oc get application openshift-gitops-operator-self-managed -n openshift-gitops

# View sync status
argocd app get openshift-gitops-operator-self-managed

# Test: Make a manual change and watch ArgoCD revert it
oc label subscription openshift-gitops-operator test=manual -n openshift-gitops-operator
# Watch it get removed by ArgoCD within a few seconds
```

## Integration with k8s-gitops-repo

To integrate this with the App of Apps pattern, add it as a component:

```bash
# In k8s-gitops-repo
mkdir -p apps/components/openshift-gitops-self-managed
```

Then reference it in cluster configs that should self-manage GitOps.

