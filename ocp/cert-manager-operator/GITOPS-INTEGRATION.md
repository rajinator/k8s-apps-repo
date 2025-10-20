# GitOps Integration - cert-manager with InstallPlanApprover

This document describes how cert-manager operator is integrated with ArgoCD and the InstallPlanApprover.

## GitOps Repository Structure

```
ocp-gitops-repo/
└── apps/
    ├── components/
    │   └── cert-manager-operator/
    │       ├── cert-manager-operator.yaml    # ArgoCD Application
    │       └── kustomization.yaml            # Component definition
    └── dev-cluster/
        └── kustomization.yaml                # Includes cert-manager component
```

## How It Works

### 1. ArgoCD Application Points to k8s-apps-repo

**File:** `ocp-gitops-repo/apps/components/cert-manager-operator/cert-manager-operator.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager-operator
  namespace: openshift-gitops
spec:
  source:
    repoURL: https://github.com/rajinator/k8s-apps-repo
    path: ocp/cert-manager-operator/overlays/with-installplan-approver
  destination:
    namespace: cert-manager
  syncPolicy:
    automated:
      selfHeal: true
```

### 2. Overlay Includes InstallPlanApprover

**File:** `k8s-apps-repo/ocp/cert-manager-operator/overlays/with-installplan-approver/`

```yaml
# Includes:
# 1. Base Subscription (with Manual approval + v1.15.0)
# 2. InstallPlanApprover CR (auto-approval)
```

### 3. Workflow

```
┌─────────────────┐
│   Git Commit    │
│  (version pin)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ArgoCD Syncs   │
│  Application    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Creates/Updates Resources:         │
│  1. Namespace                       │
│  2. OperatorGroup                   │
│  3. Subscription (Manual approval)  │
│  4. InstallPlanApprover CR          │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ OLM Creates     │
│ InstallPlan     │
│ (approved=false)│
└────────┬────────┘
         │
         ▼
┌──────────────────────────────┐
│ InstallPlanApprover Operator │
│ Watches & Approves           │
│ (< 100ms)                    │
└────────┬─────────────────────┘
         │
         ▼
┌─────────────────┐
│ cert-manager    │
│ v1.15.0         │
│ Installed       │
└─────────────────┘
```

## Deployment

### Prerequisites

1. **InstallPlanApprover Operator Running**
   ```bash
   # Clone and deploy the operator
   git clone https://github.com/rajinator/installplan-approver-operator
   cd installplan-approver-operator
   make install
   make run  # Or deploy: make deploy IMG=ghcr.io/rajinator/installplan-approver-operator:latest
   ```

2. **GitOps Repo Committed**
   ```bash
   cd ocp-gitops-repo
   git add apps/components/cert-manager-operator/
   git add apps/dev-cluster/kustomization.yaml
   git commit -m "Add cert-manager with InstallPlanApprover"
   git push
   ```

3. **App-of-Apps Applied** (if not already)
   ```bash
   oc apply -k apps/dev-cluster/
   ```

### What Gets Created

ArgoCD creates an Application that manages:

```yaml
# In OpenShift cluster:
Namespace: cert-manager
OperatorGroup: cert-manager-operator-group
Subscription: cert-manager-operator (Manual approval, v1.15.0)
InstallPlanApprover: cert-manager-approver

# OLM creates:
InstallPlan: install-xxxxx (approved by operator)
CSV: cert-manager-operator.v1.15.0
```

## Monitoring

### ArgoCD UI

1. Navigate to: `https://openshift-gitops-server-openshift-gitops.apps.<cluster>/applications`
2. Find: `cert-manager-operator` application
3. Should show: `Healthy` and `Synced`

### CLI Monitoring

```bash
# Check ArgoCD Application
oc get application cert-manager-operator -n openshift-gitops

# Check InstallPlan
oc get installplans -n cert-manager

# Check InstallPlanApprover status
oc get installplanapprovers -n cert-manager -o yaml

# Check cert-manager CSV
oc get csv -n cert-manager
```

## Version Updates via GitOps

### Step 1: Update in Git

```bash
cd k8s-apps-repo

# Edit subscription to new version
vim ocp/cert-manager-operator/base/subscription.yaml
# Change: startingCSV: cert-manager-operator.v1.16.0

# Commit
git add ocp/cert-manager-operator/base/subscription.yaml
git commit -m "Update cert-manager to v1.16.0"
git push
```

### Step 2: ArgoCD Syncs

ArgoCD detects the change and:
1. Updates the Subscription
2. OLM creates new InstallPlan
3. InstallPlanApprover automatically approves it
4. cert-manager upgrades to v1.16.0

### Step 3: Verify

```bash
# Check new CSV
oc get csv -n cert-manager

# Check InstallPlanApprover tracked the approval
oc get installplanapprovers -n cert-manager -o yaml | grep approvedCount
```

## Troubleshooting

### Application Not Syncing

```bash
# Check Application status
oc describe application cert-manager-operator -n openshift-gitops

# Force sync
argocd app sync cert-manager-operator
# Or in UI: Click "Sync" button
```

### InstallPlan Not Approved

**Check InstallPlanApprover Operator:**
```bash
# If running locally
ps aux | grep "make run"

# If deployed
oc get pods -n installplan-approver-operator-system
oc logs -f deployment/installplan-approver-operator-controller-manager \
  -n installplan-approver-operator-system
```

**Check InstallPlanApprover CR:**
```bash
oc get installplanapprovers -n cert-manager
# Should show: cert-manager-approver
```

**Check logs:**
```bash
# Should see: "Approved InstallPlan install-xxxxx in namespace cert-manager"
```

### Operator Not Installing

**Check Subscription:**
```bash
oc get subscription cert-manager-operator -n cert-manager -o yaml
```

**Check InstallPlan:**
```bash
oc get installplans -n cert-manager
# Check if approved: true
```

**Check CatalogSource:**
```bash
oc get catalogsource community-operators -n openshift-marketplace
```

## Rollback

### Via Git

```bash
# Revert to previous version
git revert <commit-hash>
git push

# ArgoCD will sync and downgrade (if OLM allows)
```

### Manual

```bash
# Delete Application (ArgoCD)
oc delete application cert-manager-operator -n openshift-gitops

# Or delete entire operator
oc delete -k ocp/cert-manager-operator/overlays/with-installplan-approver/
```

## Benefits of This Approach

✅ **Version Control**: Exact operator versions in Git  
✅ **GitOps Automation**: No manual InstallPlan approval needed  
✅ **Audit Trail**: Git history + InstallPlanApprover status  
✅ **Declarative**: Desired state in Git, actual state converges  
✅ **Repeatable**: Same manifests work across clusters  
✅ **Fast**: Auto-approval happens in < 100ms  

## Comparison

| Approach | Version Control | Automation | Scale |
|----------|----------------|------------|-------|
| **Manual Approval** | ✅ Yes (Git) | ❌ Manual patch | ❌ Poor |
| **Auto Approval (startingCSV only)** | ⚠️ Partial | ✅ Yes | ✅ Good |
| **This (GitOps + InstallPlanApprover)** | ✅ Yes (Git) | ✅ Yes | ✅ Excellent |

## Next Steps

1. **Add More Operators**: Follow same pattern for other operators
2. **Add Approval Rules**: Extend InstallPlanApprover with label filters
3. **Multi-Cluster**: Deploy to multiple clusters using ArgoCD ApplicationSets
4. **Monitoring**: Add Prometheus metrics for approval tracking

## References

- [InstallPlanApprover Operator](https://github.com/rajinator/installplan-approver-operator)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [OLM Documentation](https://olm.operatorframework.io/)

