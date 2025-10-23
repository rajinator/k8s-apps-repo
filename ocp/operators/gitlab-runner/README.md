# GitLab Runner Operator

GitLab Runner Operator for OpenShift using OLM and Kustomize.

## Features

- OLM-based operator installation
- All-namespace watch from dedicated namespace
- Manual InstallPlan approval with auto-approval job
- Kustomize-based deployment
- Optional runner instances

## Deploy

```bash
# Deploy operator
oc apply -k ocp/gitlab-runner-operator/base/

# Wait for ready
oc get csv -n gitlab-runner-operator -w
```

## Deploy Runner (Optional)

```bash
# Edit token
vi ocp/gitlab-runner-operator/optional/runner-secret.yaml

# Edit config (URL, tags, resources)
vi ocp/gitlab-runner-operator/optional/runner-instance.yaml

# Deploy
oc apply -k ocp/gitlab-runner-operator/optional/
```

## Configuration

### Update Version

Edit `base/subscription.yaml`:
```yaml
startingCSV: gitlab-runner-operator.vX.Y.Z
```

### Runner Settings

Edit `optional/runner-instance.yaml`:
- `gitlabUrl`: GitLab instance URL
- `tags`: Runner tags
- `concurrent`: Max concurrent jobs
- `resources`: CPU/memory limits

## Verify

```bash
# Check operator
oc get csv -n gitlab-runner-operator
oc get pods -n gitlab-runner-operator

# Check runners
oc get runners -A
```

## Cleanup

```bash
oc delete -k ocp/gitlab-runner-operator/optional/  # If deployed
oc delete -k ocp/gitlab-runner-operator/base/
```
