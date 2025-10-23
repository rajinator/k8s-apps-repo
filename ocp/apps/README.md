# OpenShift Applications

This directory contains Kubernetes manifests for **application workloads** deployed on OpenShift clusters.

## Structure

```
apps/
├── infrastructure/    # Infrastructure services (Gitea, Harbor, Vault, etc.)
└── workloads/        # User-facing applications
```

## Currently Empty

No workload applications are currently defined. Applications will be added as they are deployed.

## Adding Applications

When adding applications, organize them by category and include:
- Base manifests or Helm values
- Overlays for different environments (dev, staging, prod)
- README with deployment instructions

Example structure:
```
apps/
└── infrastructure/
    └── gitea/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        ├── overlays/
        │   └── production/
        │       └── kustomization.yaml
        └── README.md
```

## Related Documentation

- [Operators](../operators/README.md) - OLM operators
- [Configs](../configs/README.md) - Platform configurations

