# OpenShift Manifests

Kubernetes manifests designed for OpenShift Container Platform, organized by type: operators, configs, and applications.

## Repository Structure

```
ocp/
├── operators/      # OLM-based operators and custom operators
├── configs/        # Platform configurations and CRs
└── apps/           # Application workloads
```

## Available Components

### InstallPlan Approver Operator
- **Type**: Kubernetes Operator (not OLM-based)
- **Purpose**: Automate OLM InstallPlan approvals with version control for GitOps workflows
- **Deployment**: Kustomize
- **Features**:
  - Event-driven approval (no polling, no race conditions)
  - Version-matched approval (only approves if CSV matches Subscription's startingCSV)
  - Multi-namespace or cluster-wide support
  - GitOps-friendly automation

**Quick Deploy:**
```bash
oc apply -k operators/installplan-approver/base/
```

**Documentation:** [operators/installplan-approver/README.md](operators/installplan-approver/README.md)

### OpenShift GitOps Operator
- **Type**: OLM-based Operator
- **Purpose**: Red Hat supported ArgoCD for GitOps continuous delivery
- **Deployment**: Kustomize with OLM
- **Features**:
  - All-namespace watch from dedicated namespace
  - Manual version pinning and Automated InstallPlan approval for first deploy with job
  - Overlay for resourceHealthChecks and resource settings
  - Version-pinning aware health checks for ArgoCD

**Quick Deploy:**
```bash
oc apply -k operators/openshift-gitops/base/
```

**Documentation:** See individual operator READMEs in `operators/<operator-name>/`

### cert-manager Operator
- **Type**: OLM-based Operator
- **Purpose**: Certificate management and automation
- **Deployment**: Kustomize with OLM
- **Features**:
  - Manual version pinning with startingCSV
  - GitOps integration examples
  - InstallPlanApprover integration overlay

**Quick Deploy:**
```bash
oc apply -k operators/cert-manager/base/
```

### GitLab Runner Operator
- **Type**: OLM-based Operator
- **Purpose**: Manage GitLab CI/CD runners in OpenShift clusters
- **Deployment**: Kustomize with OLM
- **Features**:
  - All-namespace watch from dedicated namespace
  - Manual version pinning and Automated InstallPlan approval for first deploy with job
  - Optional runner instances

**Quick Deploy:**
```bash
oc apply -k operators/gitlab-runner/base/
```

## Deployment Patterns

### OLM-Based Operators

Most OpenShift operators use OLM for lifecycle management:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: operator-name
  namespace: operator-namespace
spec:
  channel: stable
  name: operator-name
  source: community-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
  startingCSV: operator-name.vX.Y.Z
```

### Kustomize Structure

All apps follow a consistent kustomize structure:

```
app-name/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── operatorgroup.yaml
│   └── subscription.yaml
└── optional/
    ├── kustomization.yaml
    └── custom-resources.yaml
```

## OpenShift Features

### Routes vs Ingress
OpenShift Routes provide additional features:
- Automatic TLS termination
- Native load balancing
- Wildcard domains

### Security Context Constraints (SCCs)
OpenShift's pod security framework:
- Restricted (default)
- Anyuid
- Privileged
- Custom SCCs

### Integrated Registry
Built-in container registry:
```bash
oc get route -n openshift-image-registry
```

## Prerequisites

- OpenShift 4.14 or later
- `oc` CLI installed and configured
- Cluster admin or namespace admin permissions
- Access to OpenShift OperatorHub

## Best Practices

### Operator Deployment
1. Use dedicated namespaces (avoid `openshift-operators`)
2. Pin operator versions with `startingCSV`
3. Use manual InstallPlan approval for production
4. Configure all-namespace watch when needed

### Resource Management
1. Set resource limits and requests
2. Use node selectors for workload placement
3. Configure pod disruption budgets
4. Apply network policies

### Security
1. Use service accounts with minimal RBAC
2. Apply appropriate SCCs
3. Use OpenShift's integrated secrets management
4. Enable pod security admission

## Common Commands

```bash
# List available operators
oc get packagemanifests

# Check operator installation
oc get csv -A

# View operator logs
oc logs -n <namespace> -l name=<operator-name>

# Check operator subscriptions
oc get subscriptions -A

# View install plans
oc get installplans -A
```

## Troubleshooting

### Operator Not Installing
```bash
# Check subscription status
oc describe subscription <name> -n <namespace>

# Check operator hub connectivity
oc get catalogsource -n openshift-marketplace

# View operator logs
oc logs -n openshift-marketplace -l olm.catalogSource=<source>
```

### InstallPlan Pending
```bash
# List pending install plans
oc get installplan -A | grep -i false

# Manually approve
oc patch installplan <name> -n <namespace> \
  --type merge --patch '{"spec":{"approved":true}}'
```

## Resources

- [OpenShift Documentation](https://docs.openshift.com/)
- [OLM Documentation](https://olm.operatorframework.io/)
- [OperatorHub.io](https://operatorhub.io/)
- [Red Hat Operators](https://catalog.redhat.com/software/operators/search)

