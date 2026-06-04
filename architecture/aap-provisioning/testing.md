# AAP Provisioning Testing Guide

This guide covers manual testing scenarios, unit tests, and integration tests for the AAP provisioning system.

For architecture details, see [README.md](README.md).

## Table of Contents

- [Manual Testing Scenarios](#manual-testing-scenarios)
- [Unit Testing](#unit-testing)
- [Integration Testing](#integration-testing)

## Manual Testing Scenarios

### Scenario 1: Successful Provision and Deprovision (AAP Direct)

This scenario tests the happy path for resource lifecycle.

**Steps:**

```bash
# 1. Create ComputeInstance
cat <<EOF | kubectl apply -f -
apiVersion: cloudkit.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: test-ci-success
  namespace: default
spec:
  name: test-vm-success
  vcpu: 2
  memory: 4096
  disk: 40
EOF

# 2. Monitor provision job status
watch -n 5 'kubectl get computeinstance test-ci-success -o json | jq ".status.jobs // [] | map(select(.type == \"provision\")) | sort_by(.timestamp) | reverse | first"'

# 3. Verify state transitions: Pending → Running → Succeeded
# Expected states over time:
#   {jobID: "9076", type: "provision", timestamp: "...", state: "Pending", message: "Job queued"}
#   {jobID: "9076", type: "provision", timestamp: "...", state: "Running", message: "Job running"}
#   {jobID: "9076", type: "provision", timestamp: "...", state: "Succeeded", message: "Job completed successfully"}

# 4. Verify CR phase: Progressing → Ready
kubectl get computeinstance test-ci-success -o jsonpath='{.status.phase}'
# Expected: Ready

# 5. Delete ComputeInstance
kubectl delete computeinstance test-ci-success

# 6. Monitor deprovision job
watch -n 5 'kubectl get computeinstance test-ci-success -o json | jq ".status.jobs // [] | map(select(.type == \"deprovision\")) | sort_by(.timestamp) | reverse | first"'

# 7. Verify deprovision state transitions: Pending → Running → Succeeded
# Expected:
#   {id: "4925", state: "Running", message: "Deprovision job triggered"}
#   {id: "4925", state: "Succeeded", message: "Deprovision completed"}

# 8. Verify cleanup and CR deletion
kubectl get computeinstance test-ci-success
# Expected: NotFound
```

**Expected Behavior:**
- Provision job completes successfully
- CR moves to Ready phase
- Deprovision job completes successfully
- CR is deleted cleanly

### Scenario 2: Deletion During Provisioning (AAP Direct)

This scenario tests the job cancellation flow when a resource is deleted while provisioning is still in progress.

**Steps:**

```bash
# 1. Create ComputeInstance
kubectl apply -f - <<EOF
apiVersion: cloudkit.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: test-ci-cancel
  namespace: default
spec:
  name: test-vm-cancel
  vcpu: 2
  memory: 4096
  disk: 40
EOF

# 2. Immediately delete (within 30 seconds, before job completes)
kubectl delete computeinstance test-ci-cancel &

# 3. Observe provision job cancellation
kubectl get computeinstance test-ci-cancel -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .state'
# Expected progression:
#   Running → Canceled

# 4. Observe deprovision triggered after cancellation
kubectl get computeinstance test-ci-cancel -o json | jq -r '.status.jobs // [] | map(select(.type == "deprovision")) | sort_by(.timestamp) | reverse | first | .jobID'
# Expected: AAP job ID (e.g., "9104")

# 5. Check operator logs for cancellation
kubectl logs -n osac-operator-system deployment/osac-operator-controller-manager | grep "cancel"
# Expected: "provision job is running, attempting to cancel"
#           "canceled provision job"

# 6. Verify final cleanup
kubectl get computeinstance test-ci-cancel
# Expected: NotFound (after deprovision completes)
```

**Expected Behavior:**
- Provision job is canceled when deletion is requested
- Operator waits for cancellation to complete
- Deprovision job is triggered after provision job reaches Canceled state
- CR is deleted cleanly

### Scenario 3: Failed Provision Job (AAP Direct)

This scenario tests error handling when a provision job fails.

**Steps:**

```bash
# 1. Create ComputeInstance with invalid configuration
# (e.g., invalid disk size, unavailable resources, network issues)
kubectl apply -f - <<EOF
apiVersion: cloudkit.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: test-ci-fail
  namespace: default
spec:
  name: test-vm-fail
  vcpu: 999  # Invalid: exceeds quota
  memory: 999999
  disk: 40
EOF

# 2. Observe job failure
kubectl get computeinstance test-ci-fail -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'
# Expected:
# {
#   "jobID": "9077",
#   "type": "provision",
#   "timestamp": "2026-02-12T10:00:00Z",
#   "state": "Failed",
#   "message": "AAP job failed: [error details from playbook]",
#   "blockDeletionOnFailure": true
# }

# 3. Check CR phase
kubectl get computeinstance test-ci-fail -o jsonpath='{.status.phase}'
# Expected: Failed

# 4. Review error details in AAP UI or via API
AAP_JOB_ID=$(kubectl get computeinstance test-ci-fail -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.result_traceback'

# 5. Attempt deletion
kubectl delete computeinstance test-ci-fail

# 6. Verify deprovision is triggered (cleanup attempt)
kubectl get computeinstance test-ci-fail -o json | jq '.status.jobs // [] | map(select(.type == "deprovision")) | sort_by(.timestamp) | reverse | first'
```

**Expected Behavior:**
- Provision job fails with error message
- CR moves to Failed phase
- Error details are visible in status
- Deprovision job is still triggered on deletion (cleanup attempt)

## Unit Testing

Unit tests validate individual components in isolation.

**Test Files:**
- `osac-operator/pkg/provisioning/provider_test.go` - Interface contract tests
- `osac-operator/pkg/provisioning/aap_provider_test.go` - AAP provider tests
- `osac-operator/pkg/aap/client_test.go` - AAP client tests

**Running Unit Tests:**

```bash
cd /path/to/osac-operator

# Run all tests
make test

# Run specific package tests
go test ./pkg/provisioning/... ./pkg/aap/... -v

# Run tests with coverage
go test ./pkg/provisioning/... ./pkg/aap/... -cover -coverprofile=coverage.out

# View coverage report
go tool cover -html=coverage.out
```

**Key Test Coverage:**
- Provider interface contract (all methods)
- Job state transitions (Pending → Running → Succeeded/Failed)
- Idempotency (multiple triggers with same job ID)
- Job cancellation logic
- Template auto-detection
- Error handling (network failures, 404s, 401s, 500s)

## Integration Testing

Integration tests validate end-to-end workflows with fake Kubernetes clients and mock AAP clients.

**Test Files:**
- `osac-operator/internal/controller/computeinstance_integration_test.go`

**Running Integration Tests:**

```bash
cd /path/to/osac-operator

# Run integration tests
go test ./internal/controller/... -v -tags=integration

# Run specific test
go test ./internal/controller/... -v -run TestComputeInstanceController_DeletionDuringProvisioning
```

**Test Scenarios Covered:**
- Successful provision → deprovision flow
- Deletion during provisioning (job cancellation)
- Failed provision job handling
- Failed deprovision job with BlockDeletionOnFailure
- Terminal state detection
- Duplicate job prevention
- Crash recovery (job ID persistence)

Integration tests use fake Kubernetes clients and mock AAP clients to verify end-to-end workflows without requiring actual AAP infrastructure.
