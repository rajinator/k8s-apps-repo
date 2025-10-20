# Repository Structure

## Overview

This repository is organized by platform to clearly separate vanilla Kubernetes and OpenShift-specific applications.

## Directory Layout

```
k8s-apps-repo/
├── README.md                           # Main repository documentation
├── STRUCTURE.md                        # This file - structure overview
├── .gitignore                          # Git ignore patterns
│
├── k8s/                                # Vanilla Kubernetes Applications
│   ├── README.md                       # [Planned] K8s platform guide
│   ├── argocd/                         # [Planned] GitOps CD
│   ├── rook-ceph/                      # [Planned] Cloud-native storage
│   └── nexus/                          # [Planned] Artifact repository
│
└── ocp/                                # OpenShift Applications
    ├── README.md                       # OCP platform guide
    ├── openshift-gitops-operator/      # ArgoCD GitOps
    │   ├── README.md                   # Full documentation
    │   ├── QUICKSTART.md               # Quick reference
    │   ├── base/                       # Operator installation
    │   │   ├── kustomization.yaml
    │   │   ├── namespace.yaml
    │   │   ├── operatorgroup.yaml
    │   │   ├── subscription.yaml
    │   │   ├── installplan-approver-rbac.yaml
    │   │   └── installplan-approver-job.yaml
    │   └── overlays/
    │       └── custom/                 # ArgoCD customization
    │           ├── kustomization.yaml
    │           └── argocd-instance.yaml
    └── gitlab-runner-operator/         # CI/CD runner operator
        ├── README.md                   # Full documentation
        ├── QUICKSTART.md               # Quick reference
        ├── base/                       # Operator installation
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   ├── operatorgroup.yaml
        │   ├── subscription.yaml
        │   ├── installplan-approver-rbac.yaml
        │   └── installplan-approver-job.yaml
        └── optional/                   # Optional runner instances
            ├── kustomization.yaml
            ├── runner-secret.yaml
            └── runner-instance.yaml
```

## Design Principles

### Platform Separation
- **k8s/**: Applications using standard Kubernetes patterns
- **ocp/**: Applications leveraging OpenShift-specific features (OLM, Routes, SCCs)

### Consistent App Structure
```
app-name/
├── README.md              # Comprehensive documentation
├── QUICKSTART.md          # Quick deployment guide
├── validate.sh            # (Optional) Validation script
├── base/                  # Core resources
│   └── kustomization.yaml
├── optional/              # Optional components
│   └── kustomization.yaml
└── .examples/             # Usage examples
```

### Deployment Methods

| Platform | Primary Methods | Examples |
|----------|----------------|----------|
| Kubernetes | Helm, Kustomize, Manifests | ArgoCD, Rook-Ceph |
| OpenShift | OLM + Kustomize, Helm, Operators | GitLab Runner Operator |

## Quick Reference

### Deploy OpenShift Operator
```bash
oc apply -k ocp/gitlab-runner-operator/base/
```

### Deploy Kubernetes App (Coming Soon)
```bash
helm install <app> <chart> -n <namespace>
# or
kubectl apply -k k8s/<app>/base/
```

## Adding New Applications

1. **Choose platform directory** (`k8s/` or `ocp/`)
2. **Create app directory** with consistent structure
3. **Add documentation** (README.md, QUICKSTART.md)
4. **Include validation** (optional validate.sh script)
5. **Update platform README** with new app entry
6. **Update main README** with app in the table

## GitOps Integration

Structure is ArgoCD-friendly:

```yaml
# Example OCP App
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab-runner-operator
spec:
  source:
    path: ocp/gitlab-runner-operator/base
    repoURL: <your-repo-url>
  destination:
    namespace: gitlab-runner-operator
```

## Current Applications

### OpenShift (ocp/)
- 🔄 OpenShift GitOps Operator (OLM, kustomize, ArgoCD with OLM health checks)
- 🏃 GitLab Runner Operator (OLM, kustomize, CI/CD runners)

### Kubernetes (k8s/)
- Applications coming soon
