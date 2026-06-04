# AAP Provisioning Operations Guide

This guide covers deployment and monitoring for AAP-based provisioning in the OSAC operator.

For architecture details, see [README.md](README.md).

## Table of Contents

- [Deploying the Operator](#deploying-the-operator)
- [Monitoring and Observability](#monitoring-and-observability)

## Deploying the Operator

The OSAC operator provisions infrastructure by launching AAP job or workflow templates over the REST API and polling job status until completion.

**Prerequisites:**
- AAP Controller accessible from the management cluster (network connectivity)
- AAP OAuth2 token with template execution permissions
- Job templates and/or workflow templates configured in AAP
- Templates accept resource JSON in `ansible_eda.event.payload` format (AAP template contract)
- Templates should NOT manage finalizers (the operator handles this)

**Deployment Steps:**

1. **Create AAP Token Secret:**
   ```bash
   kubectl create secret generic osac-aap-api-token \
     --from-literal=token="YOUR_AAP_TOKEN_HERE" \
     -n osac-operator-system
   ```

2. **Configure Operator:**
   ```yaml
   # config/manager/manager.yaml
   env:
     - name: OSAC_AAP_URL
       value: "https://aap.example.com/api/controller"
     - name: OSAC_AAP_TOKEN
       valueFrom:
         secretKeyRef:
           name: osac-aap-api-token
           key: token
     - name: OSAC_AAP_TEMPLATE_PREFIX
       value: "osac"
     - name: OSAC_AAP_STATUS_POLL_INTERVAL
       value: "30s"
   ```

3. **Deploy Operator:**
   ```bash
   kubectl apply -f config/manager/manager.yaml
   ```

**Verification:**

```bash
# Check AAP connectivity from operator pod
# OSAC_AAP_URL matches operator config; the client appends /v2/ to API paths
OSAC_AAP_URL="https://aap.example.com/api/controller"
AAP_TOKEN="your-token"

kubectl run -it --rm test-aap --image=curlimages/curl --restart=Never -- \
  curl -H "Authorization: Bearer $AAP_TOKEN" \
  "${OSAC_AAP_URL}/v2/ping/"

# Verify templates exist in AAP
curl -H "Authorization: Bearer $AAP_TOKEN" \
  "${OSAC_AAP_URL}/v2/job_templates/" | jq '.results[] | {name, id}'

curl -H "Authorization: Bearer $AAP_TOKEN" \
  "${OSAC_AAP_URL}/v2/workflow_job_templates/" | jq '.results[] | {name, id}'

# Create test ComputeInstance
cat <<EOF | kubectl apply -f -
apiVersion: osac.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: test-ci-aap
  namespace: default
spec:
  name: test-vm-aap
  vcpu: 2
  memory: 4096
  disk: 40
EOF

# Watch provision job status in CR
kubectl get computeinstance test-ci-aap -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'

# Check AAP job directly
AAP_JOB_ID=$(kubectl get computeinstance test-ci-aap -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" \
  "${OSAC_AAP_URL}/v2/jobs/$AAP_JOB_ID/" | jq '.status, .failed, .finished'

# Check operator logs for AAP API calls
kubectl logs -n osac-operator-system deployment/osac-operator-controller-manager -f | grep "AAP"
```

## Monitoring and Observability

### ComputeInstance Status Fields

The operator populates the `status` section of ComputeInstance CRs with job information.

**Status Structure:**

```yaml
status:
  phase: Progressing | Ready | Failed | Deleting

  jobs:
  - jobID: "9076"
    type: provision
    timestamp: "2026-02-12T09:55:26Z"
    state: Succeeded
    message: "Provision job succeeded"
    blockDeletionOnFailure: true

  - jobID: "9079"
    type: deprovision
    timestamp: "2026-02-12T10:12:15Z"
    state: Running
    message: "Deprovision job triggered"
    blockDeletionOnFailure: true

  conditions: [...]
```

**Querying Status:**

```bash
# Get full jobs array
kubectl get computeinstance <name> -o json | jq '.status.jobs'

# Get latest provision job
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'

# Get provision job state only
kubectl get computeinstance <name> -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .state'

# Get latest deprovision job
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "deprovision")) | sort_by(.timestamp) | reverse | first'
```

### Key Log Messages

**Provision Flow:**

```text
"triggering provisioning"
"provision job triggered"
"provision job still running"
"provision job succeeded"
"provision job failed: [error details]"
```

**Deprovision Flow:**

```text
"checking provision job before deprovision"
"provision job is running, attempting to cancel"
"canceled provision job"
"provision job already terminal"
"deprovisioning not ready, requeueing"
"deprovision job triggered"
"deprovision job succeeded"
"deprovision job failed: [error details]"
```

**Viewing Logs:**

```bash
kubectl logs -n osac-operator-system deployment/osac-operator-controller-manager -f
kubectl logs -n osac-operator-system deployment/osac-operator-controller-manager | grep "test-ci"
kubectl logs -n osac-operator-system deployment/osac-operator-controller-manager | grep "AAP"
```

### Prometheus Metrics (Future)

Future enhancements may include Prometheus metrics for job trigger counts, job duration histograms, job state distribution, API call latencies, and polling intervals.
