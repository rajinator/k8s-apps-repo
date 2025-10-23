# InstallPlan Approver Configuration

InstallPlanApprover CR configurations for different use cases.

> **Note:** This is separate from the operator deployment. The operator must be installed first.

## Structure

```
installplan-approver-config/
└── overlays/
    ├── multi-namespace/            # Centralized approval for multiple namespaces
    ├── single-namespace/           # Single namespace (for testing)
    └── cluster-wide/               # All namespaces (use with caution)
```

## Use Cases

### Multi-Namespace

Centralized approval for specific operator namespaces:

```bash
oc apply -k overlays/multi-namespace/
```

**Configuration:**
```yaml
targetNamespaces:
  - openshift-gitops-operator
  - cert-manager-operator
  - gitlab-runner-operator
```

**When to use:**
- Production environments
- Controlled operator list
- Audit compliance required

### Single Namespace (Testing)

Approve InstallPlans in one namespace only:

```bash
oc apply -k overlays/single-namespace/
```

**Configuration:**
```yaml
targetNamespaces:
  - cert-manager-operator
```

**When to use:**
- Testing new operators
- Development environments
- Isolated operator testing

### Cluster-Wide (Use with Caution)

Approve ALL InstallPlans in the cluster:

```bash
oc apply -k overlays/cluster-wide/
```

**Configuration:**
```yaml
targetNamespaces: []  # Empty = all namespaces
```

**When to use:**
- Dev/sandbox clusters
- Fully automated environments
- You trust all operators

**⚠️ Warning:** This will automatically approve ALL operator installations cluster-wide!

## Prerequisites

The InstallPlan Approver Operator must be installed first:

```bash
# Install operator from ocp/installplan-approver-operator/base/
oc apply -k ../installplan-approver-operator/base/

# Wait for operator to be ready
oc wait --for=condition=Ready pod -l app.kubernetes.io/name=installplan-approver-operator \
  -n iplan-approver-system --timeout=300s

# Then apply configuration
oc apply -k overlays/multi-namespace/
```

## Verification

Check the InstallPlanApprover CR:

```bash
# Check CR exists
oc get installplanapprovers -n iplan-approver-system

# Check CR status
oc get installplanapprovers -n iplan-approver-system -o yaml

# Check operator logs
oc logs -n iplan-approver-system -l app.kubernetes.io/name=installplan-approver-operator -f
```

## Customization

### Add More Namespaces

Edit `overlays/multi-namespace/installplanapprover.yaml`:

```yaml
spec:
  targetNamespaces:
    - openshift-gitops-operator
    - cert-manager-operator
    - gitlab-runner-operator
    - my-new-operator-namespace  # Add here
```

### Add Operator Name Filtering

For additional safety, filter by operator names:

```yaml
spec:
  targetNamespaces:
    - cert-manager-operator
  operatorNames:
    - openshift-cert-manager-operator  # Only approve this specific operator
```

### Change Approval Behavior

Disable auto-approval (manual review mode):

```yaml
spec:
  autoApprove: false
```

## GitOps Integration

This configuration is designed for ArgoCD deployment:

```yaml
# ocp-gitops-repo/apps/components/installplan-approver-config/
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: installplan-approver-config
  namespace: openshift-gitops
  labels:
    argoproj.io/sync-wave: "2"  # After operator (wave 1)
spec:
  source:
    repoURL: https://github.com/rajinator/k8s-apps-repo
    targetRevision: main
    path: ocp/installplan-approver-config/overlays/multi-namespace
  destination:
    namespace: iplan-approver-system
```

## Troubleshooting

### CR Not Created

```bash
# Check if CRD exists
oc get crd installplanapprovers.operators.bapu.cloud

# If not, install the operator first
oc apply -k ../installplan-approver-operator/base/
```

### InstallPlans Not Being Approved

```bash
# Check CR status
oc describe installplanapprovers -n iplan-approver-system

# Check operator logs
oc logs -n iplan-approver-system -l app.kubernetes.io/name=installplan-approver-operator

# Check if namespace is in targetNamespaces
oc get installplanapprovers multi-namespace-approver -n iplan-approver-system \
  -o jsonpath='{.spec.targetNamespaces}'
```

### Wrong Namespace

The CR must be in the same namespace as the operator:

```yaml
metadata:
  namespace: iplan-approver-system  # Must match operator namespace
```

## Cleanup

```bash
# Remove configuration (keeps operator running)
oc delete -k overlays/multi-namespace/

# Or remove specific CR
oc delete installplanapprover multi-namespace-approver -n iplan-approver-system
```

## References

- [InstallPlan Approver Operator](https://github.com/rajinator/installplan-approver-operator)
- [Operator Documentation](../../operators/installplan-approver/README.md)
- [OLM InstallPlans](https://olm.operatorframework.io/docs/concepts/crds/installplan/)

