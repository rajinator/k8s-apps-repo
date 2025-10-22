# K8s Apps Repository

Reference repository for deploying applications on Kubernetes and OpenShift Container Platform (OCP).

Assisted by: Cursor and claude-4-sonnet

## Structure

Applications are organized by platform and component type:

```
k8s-apps-repo/
├── k8s/              # Vanilla Kubernetes applications
│   ├── argocd/       # GitOps controller
│   ├── rook-ceph/    # Cloud-native storage
│   └── ...
└── ocp/              # OpenShift-specific applications
    ├── operators/    # OLM operator deployments
    │   ├── installplan-approver/  # InstallPlan automation
    │   ├── openshift-gitops/      # ArgoCD/GitOps
    │   ├── cert-manager/          # Certificate management
    │   ├── gitlab-runner/         # CI/CD runners
    │   └── rook-ceph/             # Cloud-native storage
    ├── configs/      # Configuration CRs for operators
    │   ├── installplan-approver/  # InstallPlanApprover CRs
    │   ├── cert-manager/          # [Planned] ClusterIssuer, Certificates
    │   └── rook-ceph/             # CephCluster and storage classes
    └── apps/         # Application deployments
```

## Platform Differences

### Kubernetes (`k8s/`)
- **Deployment Methods**: Helm, Kustomize, plain manifests
- **Operator Management**: Manual CRD installation
- **Registry**: Docker Hub, Quay.io, custom registries

### OpenShift (`ocp/`)
- **Deployment Methods**: OLM, Helm, Kustomize, Operators
- **Operator Management**: Operator Lifecycle Manager (OLM)
- **Registry**: Integrated registry, Red Hat registries, custom registries
- **Additional Features**: Routes, SCCs, integrated CI/CD

## Available Applications

### OpenShift (OCP)

#### Operators

| Application | Type | Description |
|-------------|------|-------------|
| [InstallPlan Approver Operator](ocp/operators/installplan-approver/) | Operator | Automate OLM InstallPlan approvals with version control |
| [OpenShift GitOps](ocp/operators/openshift-gitops/) | OLM | ArgoCD for GitOps CD with self-management |
| [cert-manager](ocp/operators/cert-manager/) | OLM | Red Hat cert-manager for certificate management |
| [GitLab Runner](ocp/operators/gitlab-runner/) | OLM | Manage GitLab CI/CD runners |
| [Rook Ceph](ocp/operators/rook-ceph/) | Operator | Cloud-native storage orchestrator for Ceph (v1.17.8) |

#### Configs

| Configuration | Type | Description |
|---------------|------|-------------|
| [InstallPlan Approver Config](ocp/configs/installplan-approver/) | CR | InstallPlanApprover CRs (multi-namespace, single-namespace, cluster-wide) |
| [cert-manager Config](ocp/configs/cert-manager/) | CR | ClusterIssuer and Certificate resources (placeholder) |
| [Rook Ceph Cluster](ocp/configs/rook-ceph/) | CR | Production CephCluster with storage classes (RWO block + RWX filesystem) |

### Kubernetes

> Additional applications coming soon.

## Quick Start

### Deploy on OpenShift

```bash
# Example: GitLab Runner Operator
oc apply -k ocp/operators/gitlab-runner/base/

# Example: InstallPlanApprover CR (multi-namespace)
oc apply -k ocp/configs/installplan-approver/overlays/multi-namespace/

# Example: Rook Ceph Storage (operator + cluster)
oc apply -k ocp/operators/rook-ceph/base/
# Wait for operator to be ready, then:
oc apply -k ocp/configs/rook-ceph/overlays/production/
```

### Deploy on Kubernetes

```bash
# Example: ArgoCD (coming soon)
kubectl apply -k k8s/argocd/base/
```

## Prerequisites

### For Kubernetes
- Kubernetes 1.24+
- `kubectl` CLI
- Helm 3.x (for Helm-based deployments)

### For OpenShift
- OpenShift 4.14+
- `oc` CLI
- Cluster admin or appropriate RBAC permissions

## Contributing Applications

When adding new applications:

1. **Choose the right platform and component type**
   - Use `k8s/` for vanilla Kubernetes apps
   - Use `ocp/operators/` for OLM operator deployments
   - Use `ocp/configs/` for configuration CRs that operators consume
   - Use `ocp/apps/` for application deployments

2. **Use consistent structure**
   
   **For Operators:**
   ```
   ocp/operators/operator-name/
   ├── README.md              # Comprehensive guide
   ├── base/                  # Core operator resources (Namespace, OperatorGroup, Subscription)
   │   └── kustomization.yaml
   └── overlays/              # Environment or configuration-specific overlays
       └── overlay-name/
           └── kustomization.yaml
   ```
   
   **For Configs:**
   ```
   ocp/configs/config-name/
   ├── README.md              # Documentation
   └── overlays/              # Configuration variants
       ├── multi-namespace/   # Example: Multi-namespace config
       ├── single-namespace/  # Example: Single-namespace config
       └── cluster-wide/      # Example: Cluster-wide config
   ```

3. **Document clearly**
   - Prerequisites
   - Deployment steps
   - Configuration options
   - Troubleshooting

## GitOps Integration

This repository structure is designed to work seamlessly with ArgoCD.

**Companion Repository**: [ocp-gitops-repo](https://github.com/rajinator/ocp-gitops-repo) contains the ArgoCD Application manifests that reference this repository for deploying operators and configurations.

### Example ArgoCD Applications

```yaml
# Example: Operator deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab-runner-operator
spec:
  source:
    repoURL: https://github.com/your-org/k8s-apps-repo
    path: ocp/operators/gitlab-runner/base
    targetRevision: main
  destination:
    namespace: gitlab-runner-operator

---
# Example: Configuration CR deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: installplan-approver-config
spec:
  source:
    repoURL: https://github.com/your-org/k8s-apps-repo
    path: ocp/configs/installplan-approver/overlays/multi-namespace
    targetRevision: main
  destination:
    namespace: iplan-approver-system
```

## Best Practices

### Security
- Use secrets management (Sealed Secrets, External Secrets Operator)
- Apply least privilege RBAC
- Use network policies where applicable

### Deployment
- Use Kustomize for environment-specific configs
- Pin versions in subscriptions/charts
- Test in non-production first

### Monitoring
- Include ServiceMonitor resources for Prometheus
- Add dashboards for Grafana
- Configure alerts appropriately

## Directory Structure High Level Overview

```
k8s-apps-repo/
├── README.md                    # This file
├── k8s/                         # Kubernetes apps
│   ├── README.md
│   ├── argocd/
│   ├── rook-ceph/
│   └── nexus/
└── ocp/                         # OpenShift apps
    ├── README.md
    ├── operators/               # OLM operator deployments
    │   ├── installplan-approver/
    │   │   ├── README.md
    │   │   ├── base/
    │   │   └── overlays/
    │   ├── openshift-gitops/
    │   ├── cert-manager/
    │   ├── gitlab-runner/
    │   └── rook-ceph/
    ├── configs/                 # Configuration CRs
    │   ├── installplan-approver/
    │   │   ├── README.md
    │   │   └── overlays/
    │   │       ├── multi-namespace/
    │   │       ├── single-namespace/
    │   │       └── cluster-wide/
    │   ├── cert-manager/
    │   └── rook-ceph/
    │       └── overlays/
    │           └── production/
    └── apps/                    # Application deployments
        └── README.md
```

## Related Repositories

- **[ocp-gitops-repo](https://github.com/rajinator/ocp-gitops-repo)** - ArgoCD Application manifests for GitOps-based deployment of this repository's operators and configs

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [OpenShift Documentation](https://docs.openshift.com/)
- [Kustomize Documentation](https://kustomize.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [OLM Documentation](https://olm.operatorframework.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
