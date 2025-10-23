# Quick Start

## Deploy Operator

```bash
oc apply -k ocp/gitlab-runner-operator/base/
oc get csv -n gitlab-runner-operator -w
```

## Deploy Runner

```bash
# Configure
vi ocp/gitlab-runner-operator/optional/runner-secret.yaml
vi ocp/gitlab-runner-operator/optional/runner-instance.yaml

# Deploy
oc apply -k ocp/gitlab-runner-operator/optional/
```

## Verify

```bash
oc get pods -n gitlab-runner-operator
oc get runners -A
```

## Cleanup

```bash
oc delete -k ocp/gitlab-runner-operator/optional/
oc delete -k ocp/gitlab-runner-operator/base/
```
