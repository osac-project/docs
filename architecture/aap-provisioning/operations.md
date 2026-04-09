# AAP Provisioning Operations Guide

This guide covers deployment, migration, and monitoring for the AAP provisioning system.

For architecture details, see [README.md](README.md).

## Table of Contents

- [Deploying with EDA Provider](#deploying-with-eda-provider)
- [Deploying with AAP Direct Provider](#deploying-with-aap-direct-provider)
- [Migration from EDA to AAP Direct](#migration-from-eda-to-aap-direct)
- [Monitoring and Observability](#monitoring-and-observability)

## Deploying with EDA Provider

The EDA Provider is legacy and requires an EDA service deployment. It must be explicitly enabled by setting `OSAC_PROVISIONING_PROVIDER=eda`.

**Prerequisites:**
- EDA service deployed and accessible from cloudkit-operator
- EDA webhooks configured for create/delete operations
- AAP playbooks set up to handle EDA events
- AAP playbooks manage finalizers (add on provision, remove on deprovision)
- AAP playbooks update reconciled-config-version annotation

**Deployment Steps:**

1. **Deploy EDA Service:**
   ```bash
   # Deploy EDA service to cluster
   kubectl apply -f eda-service-deployment.yaml

   # Verify EDA service is running
   kubectl get pods -n cloudkit-system | grep eda
   ```

2. **Configure Operator:**
   ```yaml
   # config/manager/manager.yaml
   env:
     - name: OSAC_PROVISIONING_PROVIDER
       value: "eda"  # Must be set explicitly; AAP direct is the default
     - name: OSAC_COMPUTE_INSTANCE_PROVISION_WEBHOOK
       value: "http://innabox-eda-service:5000/create-compute-instance"
     - name: OSAC_COMPUTE_INSTANCE_DEPROVISION_WEBHOOK
       value: "http://innabox-eda-service:5000/delete-compute-instance"
   ```

3. **Deploy Operator:**
   ```bash
   kubectl apply -f config/manager/manager.yaml
   ```

**Verification:**

```bash
# Check EDA service is running
kubectl get svc -n cloudkit-system innabox-eda-service

# Test EDA webhook endpoint (health check)
kubectl run -it --rm test-eda --image=curlimages/curl --restart=Never -- \
  curl -X POST http://innabox-eda-service:5000/health

# Create test ComputeInstance
cat <<EOF | kubectl apply -f -
apiVersion: cloudkit.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: test-ci
  namespace: default
spec:
  name: test-vm
  vcpu: 2
  memory: 4096
  disk: 40
EOF

# Watch ComputeInstance status
kubectl get computeinstance test-ci -w

# Check operator logs for webhook trigger
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep "eda-webhook"

# Verify annotation update from AAP
kubectl get computeinstance test-ci -o jsonpath='{.metadata.annotations.cloudkit\.openshift\.io/reconciled-config-version}'
```

## Deploying with AAP Direct Provider

The AAP Direct Provider communicates directly with AAP's REST API.

**Prerequisites:**
- AAP Controller accessible from management cluster (network connectivity)
- AAP OAuth2 token with template execution permissions
- Job templates and/or workflow templates configured in AAP
- Templates must accept resource JSON in `ansible_eda.event.payload` format (for compatibility with EDA)
- Templates should NOT manage finalizers (operator handles this)

**Deployment Steps:**

1. **Create AAP Token Secret:**
   ```bash
   # Create secret with AAP token
   kubectl create secret generic aap-credentials \
     --from-literal=token="YOUR_AAP_TOKEN_HERE" \
     -n cloudkit-system
   ```

2. **Configure Operator:**
   ```yaml
   # config/manager/manager.yaml
   env:
     # OSAC_PROVISIONING_PROVIDER defaults to "aap", no need to set explicitly
     - name: OSAC_AAP_URL
       value: "https://aap.example.com/api/controller"
     - name: OSAC_AAP_TOKEN
       valueFrom:
         secretKeyRef:
           name: aap-credentials
           key: token
     - name: OSAC_AAP_PROVISION_TEMPLATE
       value: "innabox-create-compute-instance"
     - name: OSAC_AAP_DEPROVISION_TEMPLATE
       value: "innabox-delete-compute-instance"
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
AAP_URL="https://aap.example.com/api/v2"
AAP_TOKEN="your-token"

kubectl run -it --rm test-aap --image=curlimages/curl --restart=Never -- \
  curl -H "Authorization: Bearer $AAP_TOKEN" \
  $AAP_URL/ping/

# Verify templates exist in AAP
curl -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/job_templates/" | jq '.results[] | {name, id}'

curl -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/workflow_job_templates/" | jq '.results[] | {name, id}'

# Create test ComputeInstance
cat <<EOF | kubectl apply -f -
apiVersion: cloudkit.openshift.io/v1alpha1
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

# Alternatively, use watch
watch -n 5 'kubectl get computeinstance test-ci-aap -o json | jq ".status.jobs // [] | map(select(.type == \"provision\")) | sort_by(.timestamp) | reverse | first"'

# Check AAP job directly
AAP_JOB_ID=$(kubectl get computeinstance test-ci-aap -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.status, .failed, .finished'

# Check operator logs for AAP API calls
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager -f | grep "AAP"
```

## Migration from EDA to AAP Direct

Migrating from EDA to AAP Direct provides better feedback and error visibility.

**When to Migrate:**
- Need real-time job status visibility
- Want to prevent orphaned cloud resources on failures
- Require job cancellation capability during deletion
- Need crash recovery guarantees (job ID persistence)
- Want full error tracebacks from failed jobs

**Migration Prerequisites:**
1. AAP templates must accept resources in EDA event format (for compatibility)
2. Update AAP playbooks to NOT manage finalizers (operator will handle)
3. AAP Controller must be accessible from operator pods
4. AAP OAuth2 token created with necessary permissions

**Migration Steps:**

1. **Prepare AAP Playbooks:**

   Remove finalizer management from playbooks:
   ```yaml
   # OLD (EDA Provider - playbook manages finalizers):
   - name: Add finalizer to ComputeInstance
     kubernetes.core.k8s:
       state: patched
       kind: ComputeInstance
       name: "{{ ansible_eda.event.payload.metadata.name }}"
       namespace: "{{ ansible_eda.event.payload.metadata.namespace }}"
       definition:
         metadata:
           finalizers:
             - cloudkit.openshift.io/compute-instance-finalizer

   # NEW (AAP Direct - operator manages finalizers):
   # Remove the above task entirely
   ```

2. **Create AAP Token Secret:**
   ```bash
   kubectl create secret generic aap-credentials \
     --from-literal=token="YOUR_AAP_TOKEN" \
     -n cloudkit-system
   ```

3. **Update Operator Deployment:**
   ```bash
   # Edit operator deployment
   kubectl edit deployment cloudkit-operator-controller-manager -n cloudkit-system

   # AAP direct is now the default, so:
   # - Remove OSAC_PROVISIONING_PROVIDER (or set to "aap")
   # - Add OSAC_AAP_* variables
   # - Remove OSAC_COMPUTE_INSTANCE_*_WEBHOOK variables
   ```

4. **Restart Operator:**
   ```bash
   kubectl rollout restart deployment/cloudkit-operator-controller-manager -n cloudkit-system
   ```

5. **Verify with Test ComputeInstance:**
   ```bash
   # Create test resource
   kubectl apply -f test-compute-instance.yaml

   # Verify job ID in status (get latest provision job)
   kubectl get computeinstance test-ci -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID'
   # Should show AAP job ID (e.g., "9076"), not "eda-webhook-1"

   # Watch job progress
   kubectl get computeinstance test-ci -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'
   ```

6. **Monitor Existing ComputeInstances:**

   ComputeInstances in steady state (Ready) will not be affected. ComputeInstances currently provisioning/deprovisioning may require manual intervention:

   ```bash
   # List ComputeInstances in non-Ready state
   kubectl get computeinstance -A --field-selector status.phase!=Ready

   # Check their latest provision job status
   kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'
   ```

**Rollback to EDA:**

If issues occur, rollback to EDA provider:

```bash
# Edit deployment
kubectl edit deployment cloudkit-operator-controller-manager -n cloudkit-system

# Change:
# - Set OSAC_PROVISIONING_PROVIDER: "eda" (AAP direct is the default, so EDA requires explicit opt-in)
# - Restore OSAC_COMPUTE_INSTANCE_*_WEBHOOK variables
# - Remove OSAC_AAP_* variables

# Restart operator
kubectl rollout restart deployment/cloudkit-operator-controller-manager -n cloudkit-system
```

**Post-Migration:**
- Update playbooks to remove annotation-based signaling if desired
- Monitor AAP API load (polling can increase API calls)
- Adjust `OSAC_AAP_STATUS_POLL_INTERVAL` if needed (30s-60s recommended)

## Monitoring and Observability

### ComputeInstance Status Fields

Both providers populate the `status` section of ComputeInstance CRs with job information.

**Status Structure:**

```yaml
status:
  phase: Progressing | Ready | Failed | Deleting

  jobs:
  - jobID: "9076"  # AAP job ID (numeric) or EDA job ID ("eda-webhook-1")
    type: provision
    timestamp: "2026-02-12T09:55:26Z"
    state: Succeeded
    message: "Provision job succeeded"
    blockDeletionOnFailure: true  # AAP: true, EDA: false

  - jobID: "9079"  # Subsequent AAP deprovision job
    type: deprovision
    timestamp: "2026-02-12T10:12:15Z"
    state: Running
    message: "Deprovision job triggered"
    blockDeletionOnFailure: true

  # Other fields...
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

# Get all jobs by type
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "provision"))'
```

### Key Log Messages (AAP Direct)

The AAP Direct Provider logs detailed messages about job lifecycle:

**Provision Flow:**
```
"triggering provisioning"                      # TriggerProvision() called
"provision job triggered"                      # AAP job created successfully
"provision job still running"                  # Polling non-terminal job
"provision job succeeded"                      # Job reached Succeeded state
"provision job failed: [error details]"        # Job failed with error
```

**Deprovision Flow:**
```
"checking provision job before deprovision"              # isReadyForDeprovision() check
"provision job is running, attempting to cancel"         # Cancellation triggered
"canceled provision job"                                 # Cancellation successful
"provision job already terminal"                         # Job already complete
"deprovisioning not ready, requeueing"                   # Waiting for cancellation
"deprovision job triggered"                              # Deprovision started
"deprovision job succeeded"                              # Cleanup complete
"deprovision job failed: [error details]"                # Cleanup failed
```

**Viewing Logs:**

```bash
# Follow operator logs
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager -f

# Filter for specific ComputeInstance
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep "test-ci"

# Filter for provision events
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep "provision"

# Filter for AAP API calls
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep "AAP"
```

### Prometheus Metrics (Future)

Future enhancements may include Prometheus metrics for:
- Job trigger counts (success/failure)
- Job duration histograms
- Job state distribution
- API call latencies
- Polling intervals
