# Red Hat cert-manager Operator for Red Hat OpenShift

Official Red Hat supported cert-manager operator deployment configuration for OpenShift using OLM.

> **Note:** This is the **Red Hat certified operator** from `redhat-operators` catalog, not the community version.

## Version

**Current Version:** v1.15.0  
**Channel:** stable-v1  
**Operator:** openshift-cert-manager-operator

## Structure

```
cert-manager-operator/
├── base/                              # Base configuration
│   ├── namespace.yaml                 # cert-manager-operator namespace
│   ├── operatorgroup.yaml             # OperatorGroup (AllNamespaces mode)
│   ├── subscription.yaml              # Subscription with Manual approval
│   └── kustomization.yaml
└── overlays/
    └── recursive-ns/                  # Overlay with recursive DNS nameservers
        ├── certManager.yaml           # CertManager CR with DNS config
        └── kustomization.yaml
```

## Architecture

```
cert-manager-operator namespace (operator pod)
    ↓ manages
cert-manager namespace (cert-manager components)
    ├── cert-manager (controller)
    ├── cert-manager-cainjector
    └── cert-manager-webhook
```

## Use Cases

### Base: Standard Deployment

Use this for standard OpenShift deployments with working DNS:

```bash
oc apply -k base/
```

**What happens:**
1. Creates `cert-manager-operator` namespace with monitoring enabled
2. Creates OperatorGroup (AllNamespaces mode)
3. Creates Subscription with `installPlanApproval: Manual`
4. InstallPlan is created and auto-approved by InstallPlanApprover operator

**Prerequisites:**
- Centralized InstallPlanApprover operator must be deployed
- See: https://github.com/rajinator/installplan-approver-operator

### Overlay: Recursive DNS Nameservers

Use this for homelab/restricted environments where cluster DNS can't resolve public DNS:

```bash
oc apply -k overlays/recursive-ns/
```

**What happens:**
1. Everything from base/ is applied
2. CertManager CR is created to configure recursive DNS nameservers (8.8.8.8, 1.1.1.1)
3. cert-manager uses these DNS servers for DNS-01 ACME challenge validation

**When to use:**
- Homelab behind NAT
- Firewall restricts DNS queries
- Cluster DNS can't resolve public records
- Split-horizon DNS environments

## Verification

After installation, verify the operator is running:

```bash
# Check subscription
oc get subscription -n cert-manager-operator openshift-cert-manager-operator

# Check InstallPlan (should be auto-approved by InstallPlanApprover)
oc get installplans -n cert-manager-operator

# Check CSV
oc get csv -n cert-manager-operator

# Check operator pods
oc get pods -n cert-manager-operator

# Check cert-manager components (deployed by operator)
oc get pods -n cert-manager

# Expected output:
# NAME                                       READY   STATUS    RESTARTS   AGE
# cert-manager-<hash>                        1/1     Running   0          3m
# cert-manager-cainjector-<hash>             1/1     Running   0          3m
# cert-manager-webhook-<hash>                1/1     Running   0          3m
```

### Verify Recursive DNS Configuration (if using overlay)

```bash
# Check CertManager CR exists
oc get certmanager cluster

# Check cert-manager deployment has DNS arguments
oc get deployment cert-manager -n cert-manager -o yaml | grep dns01-recursive

# Should show:
# - --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53
```

## Version Updates

To update cert-manager version:

1. Check available versions:
   ```bash
   oc get packagemanifest openshift-cert-manager-operator \
     -n openshift-marketplace -o yaml
   ```

2. Edit `base/subscription.yaml`:
   ```yaml
   spec:
     startingCSV: cert-manager-operator.v1.16.0  # New version
   ```

3. Commit to Git (GitOps workflow)

4. Apply changes via ArgoCD or manually:
   ```bash
   oc apply -k base/
   # or
   oc apply -k overlays/recursive-ns/
   ```

5. New InstallPlan will be created and automatically approved by InstallPlanApprover

## Troubleshooting

### InstallPlan Not Approved

**Check centralized InstallPlanApprover operator:**
```bash
oc get pods -n installplan-approver-operator
oc logs -n installplan-approver-operator -l app.kubernetes.io/name=installplan-approver-operator
```

**Check InstallPlanApprover CR exists:**
```bash
oc get installplanapprovers -A | grep cert-manager
```

**Manually approve if needed:**
```bash
oc patch installplan <installplan-name> -n cert-manager-operator \
  --type merge --patch '{"spec":{"approved":true}}'
```

### InstallPlan Not Created

**Check Subscription:**
```bash
oc get subscription -n cert-manager-operator \
  openshift-cert-manager-operator -o yaml
```

**Check CatalogSource:**
```bash
oc get catalogsource -n openshift-marketplace redhat-operators

# Check if operator is available
oc get packagemanifests | grep openshift-cert-manager-operator
```

**Check OperatorGroup:**
```bash
oc get operatorgroup -n cert-manager-operator
```

### cert-manager Components Not Deploying

**Check cert-manager namespace exists:**
```bash
oc get namespace cert-manager
```

The operator automatically creates this namespace.

**Check operator logs:**
```bash
oc logs -n cert-manager-operator \
  -l app.kubernetes.io/name=cert-manager-operator
```

### Wrong Version Installed

**Check installed CSV:**
```bash
oc get csv -n cert-manager-operator
```

**Check startingCSV in Subscription:**
```bash
oc get subscription -n cert-manager-operator \
  openshift-cert-manager-operator -o jsonpath='{.spec.startingCSV}'
```

**List available versions:**
```bash
oc get packagemanifest openshift-cert-manager-operator \
  -n openshift-marketplace -o yaml
```

## Cleanup

**Warning:** This will remove cert-manager and all certificates!

```bash
# Remove the operator
oc delete -k overlays/recursive-ns/
# or
oc delete -k base/

# Force cleanup if needed
oc delete csv -n cert-manager-operator --all
oc delete namespace cert-manager-operator
oc delete namespace cert-manager
```

## GitOps Integration

This configuration is designed for ArgoCD/OpenShift GitOps deployment. See [GITOPS-INTEGRATION.md](./GITOPS-INTEGRATION.md) for details.

## References

- [Red Hat cert-manager Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift)
- [cert-manager upstream docs](https://cert-manager.io/docs/)
- [OLM Documentation](https://olm.operatorframework.io/)
- [InstallPlanApprover Operator](https://github.com/rajinator/installplan-approver-operator)
