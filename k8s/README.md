# Kubernetes Applications

Applications designed for vanilla Kubernetes, using standard deployment methods like Helm and Kustomize.

## Available Applications

> **Note**: Applications are being added incrementally. Check back for updates.

## Planned Applications

> Applications will be added as needed.

## Deployment Patterns

### Helm-Based Applications

Most Kubernetes apps use Helm for simplified deployment:

```bash
# Add helm repo
helm repo add <repo-name> <repo-url>
helm repo update

# Install with custom values
helm install <release-name> <chart> \
  --namespace <namespace> \
  --create-namespace \
  --values values.yaml
```

### Kustomize-Based Applications

For configuration management without templating:

```bash
# Deploy with kustomize
kubectl apply -k base/

# Or with overlays
kubectl apply -k overlays/production/
```

## Structure

Each application follows a consistent structure:

```
app-name/
├── README.md              # Comprehensive guide
├── QUICKSTART.md          # Quick deployment
├── helm/                  # Helm-based deployment
│   ├── values.yaml
│   └── values-prod.yaml
├── kustomize/            # Kustomize-based deployment
│   ├── base/
│   └── overlays/
└── manifests/            # Plain YAML manifests
    └── *.yaml
```

## Prerequisites

### Required Tools
- Kubernetes 1.24 or later
- `kubectl` CLI
- `helm` 3.x (for Helm deployments)
- `kustomize` (built into kubectl)

### Cluster Requirements
- Sufficient resources (varies by app)
- Storage class available (for persistent apps)
- Ingress controller (for web-exposed apps)
- Cert-manager (recommended for TLS)

## Common Patterns

### Storage

Most apps use PersistentVolumeClaims:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

### Ingress

Expose applications via Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

### ConfigMaps and Secrets

Externalize configuration:

```bash
# Create from file
kubectl create configmap app-config --from-file=config.yaml

# Create secret
kubectl create secret generic app-secret --from-literal=password=secret
```

## Best Practices

### Resource Management
- Set resource requests and limits
- Use Horizontal Pod Autoscaler (HPA)
- Configure pod disruption budgets
- Use node affinity/anti-affinity

### Security
- Use network policies
- Apply RBAC with least privilege
- Use Pod Security Standards
- Store secrets securely (Sealed Secrets, External Secrets)

### High Availability
- Run multiple replicas
- Use pod anti-affinity
- Configure readiness/liveness probes
- Implement graceful shutdown

### Monitoring
- Add Prometheus ServiceMonitor
- Configure Grafana dashboards
- Set up alerting rules
- Log aggregation (ELK, Loki)

## Common Commands

```bash
# List all resources in namespace
kubectl get all -n <namespace>

# Check pod logs
kubectl logs -n <namespace> <pod-name> -f

# Port forward for local access
kubectl port-forward -n <namespace> svc/<service-name> 8080:80

# Execute command in pod
kubectl exec -it -n <namespace> <pod-name> -- /bin/bash

# Apply kustomization
kubectl apply -k path/to/kustomization/

# Helm operations
helm list -n <namespace>
helm upgrade <release> <chart> -n <namespace>
helm rollback <release> <revision> -n <namespace>
```

## Troubleshooting

### Pod Not Starting
```bash
# Check pod status
kubectl describe pod <pod-name> -n <namespace>

# View events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Check logs
kubectl logs <pod-name> -n <namespace> --previous
```

### Storage Issues
```bash
# Check PVC status
kubectl get pvc -n <namespace>

# Check storage class
kubectl get storageclass

# Describe PVC
kubectl describe pvc <pvc-name> -n <namespace>
```

### Network Issues
```bash
# Check service endpoints
kubectl get endpoints -n <namespace>

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <service-name>

# Check network policies
kubectl get networkpolicy -n <namespace>
```

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Hub](https://artifacthub.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [CNCF Landscape](https://landscape.cncf.io/)

