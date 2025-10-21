# GitOps Integration - Red Hat cert-manager Operator

This document describes how the Red Hat cert-manager operator is integrated with ArgoCD and the InstallPlanApprover.

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
  labels:
    operator-source: redhat-operators
    argoproj.io/sync-wave: "5"
spec:
  source:
    repoURL: https://github.com/rajinator/k8s-apps-repo
    targetRevision: v0.1.0
    path: ocp/cert-manager-operator/base
    # or for homelab/restricted DNS:
    # path: ocp/cert-manager-operator/overlays/recursive-ns
  destination:
    namespace: cert-manager-operator
  syncPolicy:
    automated:
      selfHeal: true
```

### 2. Base Configuration

**File:** `k8s-apps-repo/ocp/cert-manager-operator/base/`

```yaml
# Includes:
# 1. cert-manager-operator namespace (with monitoring)
# 2. OperatorGroup (AllNamespaces mode)
# 3. Subscription (Red Hat operator, Manual approval, v1.15.0)
```

### 3. Optional: Recursive DNS Overlay

**File:** `k8s-apps-repo/ocp/cert-manager-operator/overlays/recursive-ns/`

```yaml
# Includes base + CertManager CR
# Configures: --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53
```

### 4. Workflow

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
┌──────────────────────────────────────────┐
│  Creates/Updates Resources:              │
│  1. cert-manager-operator namespace      │
│  2. OperatorGroup                        │
│  3. Subscription (Manual approval)       │
│  4. [Optional] CertManager CR (DNS cfg)  │
└────────┬─────────────────────────────────┘
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
│ (centralized, multi-NS)      │
│ Watches & Approves           │
│ (< 100ms)                    │
└────────┬─────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ Red Hat cert-manager        │
│ Operator Installed          │
│ v1.15.0                     │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│ cert-manager components     │
│ deployed to cert-manager ns │
│ - controller                │
│ - cainjector                │
│ - webhook                   │
└─────────────────────────────┘
```

## Deployment

### Prerequisites

1. **Centralized InstallPlanApprover Operator Running**
   
   The InstallPlanApprover should be deployed as part of your GitOps bootstrap:
   
   ```yaml
   # ocp-gitops-repo/apps/dev-cluster/kustomization.yaml
   components:
     - ../components/installplan-approver-operator  # sync-wave: 1
     - ../components/cert-manager-operator          # sync-wave: 5
   ```

2. **GitOps Repo Committed and Tagged**
   ```bash
   cd ocp-gitops-repo
   git add apps/components/cert-manager-operator/
   git add apps/dev-cluster/kustomization.yaml
   git commit -m "Add Red Hat cert-manager operator"
   git tag v0.1.0
   git push origin main --tags
   ```

3. **k8s-apps-repo Committed and Tagged**
   ```bash
   cd k8s-apps-repo
   git add ocp/cert-manager-operator/
   git commit -m "Setup Red Hat cert-manager operator"
   git tag v0.1.0
   git push origin main --tags
   ```

4. **App-of-Apps Applied**
   ```bash
   oc apply -f ocp-gitops-repo/bootstrap/dev-cluster.yaml
   ```

### What Gets Created

ArgoCD creates an Application that manages:

```yaml
# In OpenShift cluster:
Namespace: cert-manager-operator
  └─ Labels: openshift.io/cluster-monitoring=true

OperatorGroup: cert-manager-operator
  └─ targetNamespaces: [] (AllNamespaces mode)

Subscription: openshift-cert-manager-operator
  └─ source: redhat-operators
  └─ channel: stable-v1
  └─ startingCSV: cert-manager-operator.v1.15.0
  └─ installPlanApproval: Manual

[Optional] CertManager CR: cluster
  └─ overrideArgs: --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53

# OLM creates (after InstallPlan approval):
InstallPlan: install-xxxxx (approved by InstallPlanApprover)
CSV: cert-manager-operator.v1.15.0

# Red Hat operator creates:
Namespace: cert-manager
Deployment: cert-manager
Deployment: cert-manager-cainjector
Deployment: cert-manager-webhook
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

# Check Subscription
oc get subscription -n cert-manager-operator

# Check InstallPlan
oc get installplans -n cert-manager-operator

# Check CSV
oc get csv -n cert-manager-operator

# Check operator pods
oc get pods -n cert-manager-operator

# Check cert-manager components
oc get pods -n cert-manager

# Check CertManager CR (if using recursive-ns overlay)
oc get certmanager cluster -o yaml
```

## Version Updates via GitOps

### Step 1: Update in Git

```bash
cd k8s-apps-repo

# Edit subscription to new version
vim ocp/cert-manager-operator/base/subscription.yaml
# Change: startingCSV: cert-manager-operator.v1.16.0

# Commit and tag
git add ocp/cert-manager-operator/base/subscription.yaml
git commit -m "Update cert-manager to v1.16.0"
git tag v0.1.1
git push origin main --tags
```

### Step 2: Update ArgoCD Application (if using git tags)

```bash
cd ocp-gitops-repo

# Update targetRevision
vim apps/components/cert-manager-operator/cert-manager-operator.yaml
# Change: targetRevision: v0.1.1

git add apps/components/cert-manager-operator/
git commit -m "Update cert-manager to k8s-apps-repo v0.1.1"
git push
```

### Step 3: ArgoCD Syncs

ArgoCD detects the change and:
1. Updates the Subscription
2. OLM creates new InstallPlan
3. InstallPlanApprover automatically approves it
4. cert-manager upgrades to v1.16.0

### Step 4: Verify

```bash
# Check new CSV
oc get csv -n cert-manager-operator

# Check pods restarted with new version
oc get pods -n cert-manager-operator
oc get pods -n cert-manager
```

## Switching Between Base and Overlay

### Use Base (No Custom DNS)

```yaml
# ocp-gitops-repo/apps/components/cert-manager-operator/cert-manager-operator.yaml
spec:
  source:
    path: ocp/cert-manager-operator/base
```

### Use Recursive NS Overlay (For Homelab)

```yaml
# ocp-gitops-repo/apps/components/cert-manager-operator/cert-manager-operator.yaml
spec:
  source:
    path: ocp/cert-manager-operator/overlays/recursive-ns
```

Commit and push. ArgoCD will sync the change automatically.

## Troubleshooting

### Application Not Syncing

```bash
# Check Application status
oc describe application cert-manager-operator -n openshift-gitops

# Check for sync errors
oc get application cert-manager-operator -n openshift-gitops -o yaml | grep -A 10 status

# Force sync via CLI
argocd app sync cert-manager-operator
# Or in UI: Click "Sync" button
```

### InstallPlan Not Approved

**Check centralized InstallPlanApprover Operator:**
```bash
# Check operator is running
oc get pods -n installplan-approver-operator

# Check logs
oc logs -n installplan-approver-operator \
  -l app.kubernetes.io/name=installplan-approver-operator -f

# Should see: "Approved InstallPlan install-xxxxx in namespace cert-manager-operator"
```

**Check InstallPlanApprover CR:**
```bash
# The CR should exist in cert-manager-operator namespace or
# be configured as multi-namespace in installplan-approver-operator namespace
oc get installplanapprovers -A | grep cert-manager

# If multi-namespace setup, check the main CR
oc get installplanapprovers -n installplan-approver-operator -o yaml
```

**Manually approve if needed:**
```bash
oc patch installplan <installplan-name> -n cert-manager-operator \
  --type merge --patch '{"spec":{"approved":true}}'
```

### Operator Not Installing

**Check Subscription:**
```bash
oc get subscription openshift-cert-manager-operator \
  -n cert-manager-operator -o yaml
```

**Check InstallPlan exists:**
```bash
oc get installplans -n cert-manager-operator
# Check if approved: true
```

**Check CatalogSource:**
```bash
oc get catalogsource redhat-operators -n openshift-marketplace

# Verify operator is available
oc get packagemanifests | grep openshift-cert-manager-operator
```

### CertManager CR Not Applied (recursive-ns overlay)

**Check if CertManager CR exists:**
```bash
oc get certmanager cluster
```

**Check cert-manager deployment args:**
```bash
oc get deployment cert-manager -n cert-manager -o yaml | grep dns01-recursive
```

If not present, the operator may not have reconciled the CertManager CR yet. Wait a few minutes or check operator logs.

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
# Delete Application (keeps resources by default)
oc delete application cert-manager-operator -n openshift-gitops

# Or delete entire operator
oc delete subscription openshift-cert-manager-operator -n cert-manager-operator
oc delete csv -n cert-manager-operator --all
```

## Benefits of This Approach

✅ **Version Control**: Exact operator versions in Git  
✅ **GitOps Automation**: No manual InstallPlan approval needed  
✅ **Centralized Approval**: One operator handles all namespaces  
✅ **Audit Trail**: Git history + InstallPlanApprover status  
✅ **Declarative**: Desired state in Git, actual state converges  
✅ **Repeatable**: Same manifests work across clusters  
✅ **Fast**: Auto-approval happens in < 100ms  
✅ **Red Hat Supported**: Official operator with support  

## Next Steps

1. **Create ClusterIssuers**: Configure Let's Encrypt or other CA
2. **Create Certificates**: For ingress routes, API server, etc.
3. **Configure monitoring**: Operator namespace already has monitoring enabled
4. **Add more operators**: Follow same pattern for other operators
5. **Multi-cluster**: Deploy to multiple clusters using ArgoCD ApplicationSets

## References

- [InstallPlanApprover Operator](https://github.com/rajinator/installplan-approver-operator)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Red Hat cert-manager Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift)
- [OLM Documentation](https://olm.operatorframework.io/)
