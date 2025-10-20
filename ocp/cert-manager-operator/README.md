# Cert Manager Operator

Deployment configuration for cert-manager operator on OpenShift using OLM.

## Version

**Current Version:** v1.15.0

## Structure

```
cert-manager-operator/
├── base/                              # Base configuration
│   ├── namespace.yaml                 # Operator namespace
│   ├── operatorgroup.yaml             # OperatorGroup (all namespaces)
│   ├── subscription.yaml              # Subscription with Manual approval
│   └── kustomization.yaml
└── overlays/
    └── installplanapprover-test/      # Test overlay with InstallPlanApprover
        ├── installplanapprover.yaml   # InstallPlanApprover CR
        └── kustomization.yaml
```

## Use Cases

### Base: Manual Approval (Version Control)

Use this when you want precise version control with manual InstallPlan approval:

```bash
oc apply -k base/
```

**What happens:**
1. Creates `cert-manager-operator` namespace
2. Creates OperatorGroup
3. Creates Subscription with `installPlanApproval: Manual`
4. InstallPlan is created but **NOT approved** (manual intervention required)

**To approve manually:**
```bash
# List InstallPlans
oc get installplans -n cert-manager-operator

# Approve specific InstallPlan
oc patch installplan <installplan-name> -n cert-manager-operator \
  --type merge --patch '{"spec":{"approved":true}}'
```

### Overlay: With InstallPlanApprover (GitOps + Automation)

Use this to test the InstallPlanApprover operator:

```bash
oc apply -k overlays/installplanapprover-test/
```

**What happens:**
1. Everything from base/ is applied
2. InstallPlanApprover CR is created
3. InstallPlanApprover operator automatically approves the InstallPlan
4. cert-manager operator installs automatically

**Prerequisites:**
- InstallPlanApprover operator must be deployed and running
- See: https://github.com/rajinator/installplan-approver-operator

## Testing the InstallPlanApprover

### Step 1: Deploy InstallPlanApprover Operator

```bash
# Clone and setup InstallPlanApprover operator
git clone https://github.com/rajinator/installplan-approver-operator
cd installplan-approver-operator

# Install CRD
make install

# Run locally (easiest for testing)
make run

# Or deploy to cluster
# make deploy IMG=<your-image>
```

### Step 2: Deploy cert-manager with InstallPlanApprover

In another terminal:

```bash
cd k8s-apps-repo

# Apply the overlay
oc apply -k ocp/cert-manager-operator/overlays/installplanapprover-test/
```

### Step 3: Watch It Work

```bash
# Watch InstallPlans (should get approved automatically)
oc get installplans -n cert-manager-operator -w

# Watch InstallPlanApprover status
oc get installplanapprovers -n cert-manager-operator cert-manager-approver -o yaml

# Check cert-manager operator deployment
oc get csv -n cert-manager-operator
```

### Step 4: Check Logs

```bash
# If running locally (Terminal 1)
# Logs appear in the terminal where you ran 'make run'

# If deployed to cluster
oc logs -f deployment/installplan-approver-operator-controller-manager \
  -n installplan-approver-operator-system
```

## Expected Behavior

### Without InstallPlanApprover

```bash
oc apply -k base/

# InstallPlan created but not approved
oc get installplans -n cert-manager-operator
# NAME                            CSV                                APPROVAL    APPROVED
# install-xxxxx                   cert-manager-operator.v1.15.0      Manual      false

# Manual approval needed
oc patch installplan install-xxxxx -n cert-manager-operator \
  --type merge --patch '{"spec":{"approved":true}}'
```

### With InstallPlanApprover

```bash
oc apply -k overlays/installplanapprover-test/

# InstallPlan automatically approved (within ~100ms)
oc get installplans -n cert-manager-operator
# NAME                            CSV                                APPROVAL    APPROVED
# install-xxxxx                   cert-manager-operator.v1.15.0      Manual      true

# Check approver status
oc get installplanapprovers cert-manager-approver -n cert-manager-operator -o yaml
# status:
#   approvedCount: 1
#   lastApprovedPlan: cert-manager-operator/install-xxxxx
#   lastApprovedTime: 2025-10-20T...
```

## Version Updates

To update cert-manager version:

1. Edit `base/subscription.yaml`:
   ```yaml
   spec:
     startingCSV: cert-manager-operator.v1.16.0  # New version
   ```

2. Commit to Git (GitOps workflow)

3. Apply changes:
   ```bash
   oc apply -k overlays/installplanapprover-test/
   ```

4. New InstallPlan will be created and automatically approved

## Troubleshooting

### InstallPlan Not Approved

**Check InstallPlanApprover is running:**
```bash
# If running locally
ps aux | grep "make run"

# If deployed
oc get pods -n installplan-approver-operator-system
```

**Check InstallPlanApprover CR exists:**
```bash
oc get installplanapprovers -n cert-manager-operator
```

**Check InstallPlanApprover logs:**
```bash
# Look for approval messages
# Should see: "Approved InstallPlan install-xxxxx in namespace cert-manager-operator"
```

### InstallPlan Not Created

**Check Subscription:**
```bash
oc get subscription -n cert-manager-operator cert-manager-operator -o yaml
```

**Check CatalogSource:**
```bash
oc get catalogsource -n openshift-marketplace community-operators
```

**Check operator availability:**
```bash
oc get packagemanifests | grep cert-manager-operator
```

### Wrong Version Installed

**Check startingCSV in Subscription:**
```bash
oc get subscription -n cert-manager-operator cert-manager-operator -o yaml | grep startingCSV
```

**List available versions:**
```bash
oc get packagemanifests cert-manager-operator -o yaml
```

## Cleanup

```bash
# Remove cert-manager operator
oc delete -k overlays/installplanapprover-test/

# Or just the base
oc delete -k base/

# Force cleanup if needed
oc delete namespace cert-manager-operator
```

## References

- [Cert Manager Operator](https://github.com/cert-manager/cert-manager-operator)
- [OLM Documentation](https://olm.operatorframework.io/)
- [InstallPlanApprover Operator](../../installplan-approver-operator/)

