# K8s Apps Repository

Reference repository for deploying applications on Kubernetes and OpenShift Container Platform (OCP).

Assisted by: Cursor and claude-4-sonnet

## Structure

Applications are organized by platform:

```
k8s-apps-repo/
├── k8s/              # Vanilla Kubernetes applications
│   ├── argocd/       # GitOps controller
│   ├── rook-ceph/    # Cloud-native storage
│   └── ...
└── ocp/              # OpenShift-specific applications
    ├── gitlab-runner-operator/  # CI/CD runner operator
    └── ...
```

## Platform Differences

### Kubernetes (`k8s/`)
- **Deployment Methods**: Helm, Kustomize, plain manifests
- **Operator Management**: Manual CRD installation
- **Registry**: Docker Hub, Quay.io, custom registries

### OpenShift (`ocp/`)
- **Deployment Methods**: OLM, Helm, Kustomize, Operators
- **Operator Management**: Operator Lifecycle Manager (OLM)
- **Registry**: Integrated registry, Red Hat registries
- **Additional Features**: Routes, SCCs, integrated CI/CD

## Available Applications

### OpenShift (OCP)

| Application | Type | Description |
|-------------|------|-------------|
| [InstallPlan Approver Operator](ocp/installplan-approver-operator/) | Operator | Automate OLM InstallPlan approvals with version control |
| [OpenShift GitOps Operator](ocp/openshift-gitops-operator/) | OLM | ArgoCD for GitOps CD with version-pinning aware health checks |
| [cert-manager Operator](ocp/cert-manager-operator/) | OLM | Certificate management with GitOps integration |
| [GitLab Runner Operator](ocp/gitlab-runner-operator/) | OLM | Manage GitLab CI/CD runners |

### Kubernetes

> Additional applications coming soon.

## Quick Start

### Deploy on OpenShift

```bash
# Example: GitLab Runner Operator
oc apply -k ocp/gitlab-runner-operator/base/
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

1. **Choose the right platform directory**
   - Use `k8s/` for vanilla Kubernetes apps
   - Use `ocp/` for OpenShift-specific features (OLM, Routes, etc.)

2. **Use consistent structure**
   ```
   app-name/
   ├── README.md              # Comprehensive guide
   ├── QUICKSTART.md          # TL;DR commands
   ├── base/                  # Core resources
   │   └── kustomization.yaml
   ├── optional/              # Optional components
   │   └── kustomization.yaml
   └── .examples/             # Usage examples
   ```

3. **Document clearly**
   - Prerequisites
   - Deployment steps
   - Configuration options
   - Troubleshooting

## GitOps Integration

This repository structure is designed to work seamlessly with ArgoCD:

```yaml
# Example ArgoCD Application for OCP
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab-runner-operator
spec:
  source:
    repoURL: https://github.com/your-org/k8s-apps-repo
    path: ocp/gitlab-runner-operator/base
    targetRevision: main
  destination:
    namespace: gitlab-runner-operator
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
    └── gitlab-runner-operator/
        ├── README.md
        ├── QUICKSTART.md
        ├── base/
        ├── optional/
        └── .examples/
```

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [OpenShift Documentation](https://docs.openshift.com/)
- [Kustomize Documentation](https://kustomize.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [OLM Documentation](https://olm.operatorframework.io/)
