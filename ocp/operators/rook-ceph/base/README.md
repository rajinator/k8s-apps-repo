# Rook Ceph Operator v1.17.8

Rook is a cloud-native storage orchestrator for Kubernetes, providing the platform, framework, and support for Ceph storage to natively integrate with Kubernetes.

## Overview

This deployment uses the **OpenShift-specific** Rook operator that leverages SecurityContextConstraints (SCC) instead of PodSecurityPolicies.

### Components

1. **CRDs** (`crds.yaml`) - Custom Resource Definitions
   - CephCluster, CephBlockPool, CephFilesystem, CephObjectStore, etc.

2. **Common** (`common.yaml`) - Shared Resources
   - ServiceAccounts (rook-ceph-system, rook-ceph-osd, etc.)
   - ClusterRoles and ClusterRoleBindings
   - ConfigMaps and Services

3. **Operator** (`operator-openshift.yaml`) - OpenShift Operator
   - Deployment of rook-ceph-operator
   - Uses `privileged` SCC for container runtime access
   - Watches for CephCluster CRs

## Deployment

### Prerequisites

- OpenShift 4.x cluster
- Available raw block devices or PVs for OSDs
- Minimum 3 worker nodes (for production)

### Order

1. Operator deployment (this base)
2. CephCluster CR (see `ocp/configs/rook-ceph/`)

## GitOps Integration

Deployed via ArgoCD Application:
- Path: `ocp/operators/rook-ceph/base`
- Sync Wave: 5
- Namespace: `rook-ceph` (auto-created)

## References

- [Rook Ceph Documentation](https://rook.io/docs/rook/v1.17/)
- [OpenShift Storage Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/index)
- [GitHub Repository](https://github.com/rook/rook/tree/v1.17.8)

