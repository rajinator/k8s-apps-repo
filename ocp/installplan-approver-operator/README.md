# InstallPlan Approver Operator

Kubernetes operator for automatically approving OLM (Operator Lifecycle Manager) InstallPlans with version control and GitOps integration.

## Overview

This operator automates InstallPlan approvals while maintaining precise version control through Subscription manifests. It's designed for GitOps workflows where operator versions are managed in Git via `installPlanApproval: Manual` and `startingCSV`.

**Key Feature:** Only approves InstallPlans whose CSV exactly matches the Subscription's `startingCSV`, preventing accidental auto-upgrades.

## Why Use This Operator?

### Problem
- Manual `installPlanApproval: Manual` requires manual `kubectl patch` commands
- CronJobs polling for InstallPlans create race conditions and miss timing
- GitOps pipelines blocked by manual approval steps
- Managing hundreds of operators across clusters becomes unmanageable

### Solution
- **Event-driven**: Uses informers to react immediately when InstallPlans are created
- **Version-matched**: Only approves if CSV matches Subscription's `startingCSV`
- **GitOps-friendly**: Fully automated approval with Git as the source of truth
- **Efficient**: Minimal API load with intelligent requeue backoff

## Repository Structure

```
ocp/installplan-approver-operator/
├── README.md
├── base/                           # Operator only (no CRs)
│   └── kustomization.yaml          # References operator from GitHub
└── overlays/                       # Complete deployments (operator + CR)
    ├── single-namespace/           # For one namespace
    │   ├── kustomization.yaml
    │   └── installplanapprover.yaml
    ├── multi-namespace/            # For multiple namespaces (recommended)
    │   ├── kustomization.yaml
    │   └── installplanapprover.yaml
    └── cluster-wide/               # For all namespaces (use with caution)
        ├── kustomization.yaml
        └── installplanapprover.yaml
```

## Quick Start

### Deployment Options

**Option 1: All-in-One Overlay (Recommended)**
```bash
# Deploy operator + CR together using an overlay
oc apply -k ocp/installplan-approver-operator/overlays/multi-namespace/
```

**Option 2: Base Only (For custom CRs)**
```bash
# Deploy operator only, then create your own InstallPlanApprover CRs
oc apply -k ocp/installplan-approver-operator/base/
oc apply -f my-custom-approver.yaml
```

**Option 3: Direct from Operator Repository**
```bash
# Deploy from the operator's GitHub repository
oc apply -k 'github.com/rajinator/installplan-approver-operator/config/default?ref=v0.1.0'
# Then create your InstallPlanApprover CRs separately
```

## Overlays

Pre-configured overlays for common deployment patterns. Each overlay includes both the operator and an InstallPlanApprover CR.

### Single Namespace

Deploy operator + CR for approving InstallPlans in one namespace:

```bash
oc apply -k ocp/installplan-approver-operator/overlays/single-namespace/
```

**What it deploys:**
- Operator in `iplan-approver-system` namespace
- InstallPlanApprover CR in `cert-manager` namespace
- Approves only `cert-manager` namespace

**Use case:** Testing, isolated operator deployments

### Multi-Namespace

Deploy operator + CR for approving InstallPlans across multiple namespaces:

```bash
oc apply -k ocp/installplan-approver-operator/overlays/multi-namespace/
```

**What it deploys:**
- Operator in `iplan-approver-system` namespace
- InstallPlanApprover CR in `operators` namespace
- Approves: `openshift-gitops-operator`, `cert-manager`, `gitlab-runner-operator`

**Use case:** Multiple team namespaces, environment-specific operators

### Cluster-Wide

Deploy operator + CR for approving InstallPlans across all namespaces:

```bash
oc apply -k ocp/installplan-approver-operator/overlays/cluster-wide/
```

**What it deploys:**
- Operator in `iplan-approver-system` namespace
- InstallPlanApprover CR in `iplan-approver-system` namespace
- Approves: **ALL namespaces** (targetNamespaces: [])

**Use case:** Platform operators, cluster-admin managed operators  
**Warning:** Use `operatorNames` filter for safety!

## Configuration

### Basic Configuration

```yaml
apiVersion: operators.bapu.cloud/v1alpha1
kind: InstallPlanApprover
metadata:
  name: my-approver
  namespace: operators
spec:
  # Enable automatic approval
  autoApprove: true
  
  # Target specific namespaces (empty = all namespaces)
  targetNamespaces:
    - openshift-operators
    - my-operators
  
  # Optional: Filter by operator names
  operatorNames:
    - prometheus
    - cert-manager
```

### Advanced Configuration

```yaml
apiVersion: operators.bapu.cloud/v1alpha1
kind: InstallPlanApprover
metadata:
  name: production-approver
  namespace: operators
spec:
  autoApprove: true
  
  # Watch all namespaces
  targetNamespaces: []
  
  # Approve only specific operators
  operatorNames:
    - prometheus
    - grafana-operator
    - cert-manager
    - openshift-gitops-operator
```

## How It Works

### Version Matching Logic

1. **InstallPlan Created**: OLM creates an InstallPlan for an operator
2. **Operator Watches**: InstallPlanApprover operator receives event immediately
3. **Find Owner Subscription**: Operator finds the Subscription that owns the InstallPlan
4. **Compare Versions**: Compares InstallPlan's CSV with Subscription's `startingCSV`
5. **Approve if Match**: Only approves if versions exactly match
6. **Skip if Different**: Skips approval for upgrade InstallPlans (non-matching CSV)

This prevents accidental auto-approval of upgrades while still automating the initial installation.

### Example Scenario

**Subscription in Git:**
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: prometheus-operator
spec:
  channel: stable
  name: prometheus
  installPlanApproval: Manual
  startingCSV: prometheus-operator.v0.68.0  # ← Pinned version
```

**What Happens:**
- ✅ InstallPlan for `prometheus-operator.v0.68.0` → **Auto-approved**
- ❌ InstallPlan for `prometheus-operator.v0.69.0` → **Not approved** (requires Git update)

**To Upgrade:**
1. Update `startingCSV` in Git to `prometheus-operator.v0.69.0`
2. Git sync (ArgoCD/Flux) updates Subscription
3. New InstallPlan created for v0.69.0
4. Operator automatically approves (now matches `startingCSV`)

## Integration with ArgoCD

### Recommended Approach

For production GitOps environments, deploy the operator and CRs separately:

**Step 1: Deploy the operator (one time, cluster-wide)**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: installplan-approver-operator
  namespace: openshift-gitops
  labels:
    app-type: operator
    argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/rajinator/k8s-apps-repo
    targetRevision: HEAD
    path: ocp/installplan-approver-operator/overlays/multi-namespace
  destination:
    name: in-cluster
    namespace: iplan-approver-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Step 2: Configure operators to use centralized approval**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager-operator
  namespace: openshift-gitops
  labels:
    app-type: operator
    approval-mode: centralized  # ← Indicates centralized approval
    argoproj.io/sync-wave: "5"  # ← Deploy after approver operator
spec:
  project: default
  source:
    repoURL: https://github.com/rajinator/k8s-apps-repo
    targetRevision: HEAD
    path: ocp/cert-manager-operator/base
  destination:
    name: in-cluster
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

### Custom Resource Health Checks

Add these health checks to your ArgoCD instance to properly track version-pinned operators:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  resourceHealthChecks:
    # Mark unapproved InstallPlans as "Suspended" (not Progressing)
    - group: operators.coreos.com
      kind: InstallPlan
      check: |
        hs = {}
        if obj.spec.approved == false then
          hs.status = "Suspended"
          hs.message = "InstallPlan not approved (waiting for version match or manual approval)"
          return hs
        end
        -- (rest of health check logic)
    
    # Mark Subscriptions as "Healthy" if installedCSV matches startingCSV
    - group: operators.coreos.com
      kind: Subscription
      check: |
        hs = {}
        if obj.status ~= nil and obj.status.installedCSV ~= nil then
          if obj.spec.startingCSV ~= nil and obj.status.installedCSV == obj.spec.startingCSV then
            hs.status = "Healthy"
            hs.message = "Installed: " .. obj.status.installedCSV
            return hs
          end
        end
        -- (rest of health check logic)
```

**Full health check examples:** See [openshift-gitops-operator custom overlay](../openshift-gitops-operator/overlays/custom/)

### Complete Example

See the [cert-manager-operator](../cert-manager-operator/) directory for a complete working example with:
- Base manifests with version pinning
- ArgoCD Application manifests
- Integration with centralized InstallPlan approval
- Custom health checks for accurate status reporting

## Monitoring

### Check Operator Status

```bash
# View operator logs
kubectl logs -n iplan-approver-system -l control-plane=controller-manager -f

# Check InstallPlanApprover status
kubectl get installplanapprovers -A
kubectl describe installplanapprover <name> -n <namespace>
```

### InstallPlanApprover Status

```yaml
status:
  approvedCount: 5
  lastApprovedPlan: install-xyz123
  lastApprovedTime: "2025-10-21T12:34:56Z"
```

## Troubleshooting

### InstallPlan Not Being Approved

**Check 1: Operator Running?**
```bash
kubectl get pods -n iplan-approver-system
```

**Check 2: InstallPlanApprover Created?**
```bash
kubectl get installplanapprovers -A
```

**Check 3: Namespace Targeted?**
```bash
kubectl get installplanapprover <name> -n <namespace> -o yaml
# Check spec.targetNamespaces includes the InstallPlan's namespace
```

**Check 4: Version Matches?**
```bash
# Check Subscription's startingCSV
kubectl get subscription <name> -n <namespace> -o jsonpath='{.spec.startingCSV}'

# Check InstallPlan's CSV
kubectl get installplan <name> -n <namespace> -o jsonpath='{.spec.clusterServiceVersionNames[0]}'

# These must match exactly for approval
```

**Check 5: Operator Logs**
```bash
kubectl logs -n iplan-approver-system -l control-plane=controller-manager --tail=100
```

### InstallPlan Approved But Operator Not Installing

This is an OLM issue, not the InstallPlanApprover operator:
```bash
# Check InstallPlan status
kubectl get installplan <name> -n <namespace> -o yaml

# Check CSV status
kubectl get csv -n <namespace>

# Check operator pod logs
kubectl logs -n <namespace> -l app=<operator-name>
```

## Comparison with Alternatives

| Method | Event-Driven | Version Control | Race Conditions | API Load |
|--------|-------------|-----------------|-----------------|----------|
| **InstallPlanApprover Operator** | ✅ Yes | ✅ Yes (CSV match) | ❌ None | ⚡ Minimal |
| Manual `kubectl patch` | ❌ No | ✅ Yes | ❌ None | ⚡ None |
| CronJob polling | ❌ No | ❌ No | ⚠️ Possible | ⚠️ High |
| Job (one-time) | ❌ No | ❌ No | ⚠️ Possible | ⚡ Low |

## Resources

- **Operator Repository**: https://github.com/rajinator/installplan-approver-operator
- **Container Images**: `ghcr.io/rajinator/installplan-approver-operator`
- **Documentation**: 
  - [Deployment Guide](https://github.com/rajinator/installplan-approver-operator/blob/main/DEPLOY.md)
  - [Release Notes](https://github.com/rajinator/installplan-approver-operator/releases)
- **OLM Documentation**: https://olm.operatorframework.io/

