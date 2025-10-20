# InstallPlan Approval Mechanisms

Separate from the base operator resources, these provide ways to approve OLM InstallPlans.

## Structure

```
approval/
├── bootstrap/          # One-time Job for initial install
│   ├── installplan-approver-job.yaml
│   ├── installplan-approver-rbac.yaml
│   └── kustomization.yaml
└── continuous/         # CronJob for ongoing approvals
    ├── installplan-approver-cronjob.yaml
    └── kustomization.yaml (references RBAC from bootstrap)
```

## When to Use

### Bootstrap (One-Time)

```bash
# During initial manual installation
oc apply -k base/
oc apply -k approval/bootstrap/
```

**Purpose:** Approve the first InstallPlan when bootstrapping a cluster manually.

**Lifecycle:** Job runs once, approves pending InstallPlan, stays completed.

### Continuous (Automatic)

```bash
# For clusters with auto-approval via GitOps
# (Included automatically in overlays/with-auto-approval)
```

**Purpose:** Automatically approve InstallPlans for operator upgrades.

**Lifecycle:** CronJob runs every 5 minutes, checks for pending InstallPlans.

## Why Separate?

1. **Clean base**: Operator resources don't mix with approval logic
2. **Explicit choice**: Include approval only when needed
3. **No directory exclusions**: Overlays cleanly reference what they need
4. **Clear intent**: `approval/` directory makes it obvious what these do

## Comparison

| Aspect | bootstrap/ | continuous/ |
|--------|-----------|-------------|
| Type | Job | CronJob |
| Runs | Once | Every 5 min |
| Use Case | Manual install | GitOps auto-approval |
| Managed by | Manual (oc apply) | ArgoCD (optional) |
| Cleanup | Manual delete | TTL auto-cleanup |

