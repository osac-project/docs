# AAP Provisioning Troubleshooting Guide

This guide covers common issues, debugging commands, and recovery procedures for the AAP provisioning system.

For architecture details, see [README.md](README.md).

## Table of Contents

- [Common Issues](#common-issues)
- [Debugging Commands](#debugging-commands)
- [Recovery Procedures](#recovery-procedures)

## Common Issues

### EDA Provider Issues

| Issue | Symptoms | Root Cause | Solution |
|-------|----------|------------|----------|
| Webhook not triggered | Provision job ID never appears in status | EDA service unreachable, network issues | Check EDA service logs: `kubectl logs -n cloudkit-system deployment/eda-service`. Verify webhook URL is correct. Test connectivity from operator pod. |
| AAP finalizer not set | CR stuck in Deleting, finalizer never removed | AAP playbook not adding finalizer on successful provision | Update AAP playbook to add `cloudkit.openshift.io/compute-instance-finalizer` to `metadata.finalizers`. |
| Annotation not updated | CR stays in Progressing forever | AAP playbook not setting `reconciled-config-version` annotation | Update AAP playbook to set `cloudkit.openshift.io/reconciled-config-version` annotation to match desired version. |
| EDA 500 error | Webhook call returns 500 | EDA rulebook error, AAP template misconfiguration | Check EDA logs for rulebook errors. Verify AAP job template exists and is accessible. |

### AAP Direct Provider Issues

| Issue | Symptoms | Root Cause | Solution |
|-------|----------|------------|----------|
| Authentication failure | Error: "401 Unauthorized" | Invalid or expired AAP token | Verify AAP token in secret: `kubectl get secret aap-credentials -o jsonpath='{.data.token}' \| base64 -d`. Test token with curl: `curl -H "Authorization: Bearer $TOKEN" $AAP_URL/ping/`. Regenerate token if expired. |
| Job not found | Error: "404 Not Found" when polling job | AAP purges old jobs based on retention policy | AAP automatically purges completed jobs after retention period. Operator treats 404 as terminal state and proceeds. Adjust AAP job retention if needed. |
| Template not found | Error: "Template 'foo' not found" | Template name mismatch or template doesn't exist | List templates: `curl -H "Authorization: Bearer $TOKEN" $AAP_URL/job_templates/`. Verify `OSAC_AAP_PROVISION_TEMPLATE` matches AAP template name exactly (case-sensitive). |
| Cancellation not allowed | Error: "405 Method Not Allowed" when canceling | Job already in terminal state | Job reached terminal state between check and cancel. Operator proceeds with deprovision. This is normal and not an error. |
| SSL certificate error | Error: "x509: certificate signed by unknown authority" | AAP uses self-signed certificate or custom CA | Add CA certificate to operator deployment. Mount CA cert as volume and set `SSL_CERT_FILE` environment variable. Or disable SSL verification (not recommended for production). |
| Poll interval too aggressive | AAP API rate-limiting errors | Poll interval set too low | Increase `OSAC_AAP_STATUS_POLL_INTERVAL` to 30s or 60s. Monitor AAP API load. |

## Debugging Commands

### General Debugging

```bash
# View full ComputeInstance status
kubectl get computeinstance <name> -o yaml

# Check latest provision job status
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'

# Check latest deprovision job status
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "deprovision")) | sort_by(.timestamp) | reverse | first'

# Check CR phase
kubectl get computeinstance <name> -o jsonpath='{.status.phase}'

# Check finalizers
kubectl get computeinstance <name> -o jsonpath='{.metadata.finalizers}'

# Check annotations
kubectl get computeinstance <name> -o jsonpath='{.metadata.annotations}' | jq
```

### Controller Logs

```bash
# Follow controller logs
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager -f

# Filter logs for specific ComputeInstance
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep "test-ci"

# Filter for errors
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep -i error

# Filter for provision events
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager | grep provision

# Get recent logs (last 100 lines)
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager --tail=100
```

### AAP API Debugging

```bash
# Set variables
AAP_URL="https://aap.example.com/api/v2"
AAP_TOKEN="your-token-here"

# Test connectivity
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/ping/"

# List job templates
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/job_templates/" | jq '.results[] | {name, id}'

# List workflow templates
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/workflow_job_templates/" | jq '.results[] | {name, id}'

# Query specific job
AAP_JOB_ID=$(kubectl get computeinstance <name> -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq

# Check job status only
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.status, .failed, .finished'

# Get job error details
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.result_traceback'

# Check if job can be canceled
curl -X POST -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/cancel/"
# Response codes:
# 202 = cancellation initiated
# 405 = job already terminal (cannot cancel)
# 404 = job not found

# List recent jobs
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/?order_by=-id&page_size=10" | jq '.results[] | {id, name, status, finished}'
```

### Network Debugging

```bash
# Test AAP connectivity from operator pod
kubectl exec -it -n cloudkit-system deployment/cloudkit-operator-controller-manager -- sh

# Inside pod:
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/ping/"

# Test DNS resolution
nslookup aap.example.com

# Test TLS handshake
openssl s_client -connect aap.example.com:443 -servername aap.example.com

# Exit pod
exit
```

## Recovery Procedures

### Stuck ComputeInstance in Progressing

**Symptoms:**
- CR stuck in Progressing phase
- Provision job shows Running state for extended period
- No updates to CR status

**Diagnosis:**

```bash
# Check latest provision job state
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first'

# If job ID exists, check AAP directly
AAP_JOB_ID=$(kubectl get computeinstance <name> -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.status'
```

**Recovery:**

```bash
# If AAP job succeeded but CR not updated:
# 1. Check if VM/resources were actually created
# 2. Manually update CR status if needed
# 3. Or delete and recreate

# If AAP job failed:
# 1. Check error details in AAP
AAP_JOB_ID=$(kubectl get computeinstance <name> -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.result_traceback'

# 2. Fix underlying issue (quota, network, etc.)
# 3. Delete and recreate ComputeInstance

# If safe to force delete (no cloud resources created):
kubectl patch computeinstance <name> -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete computeinstance <name>
```

### Failed Deprovision Job (BlockDeletionOnFailure=true)

**Symptoms:**
- CR stuck in Deleting phase
- Deprovision job shows Failed state
- Finalizer prevents CR deletion

**Diagnosis:**

```bash
# Check latest deprovision job status
kubectl get computeinstance <name> -o json | jq '.status.jobs // [] | map(select(.type == "deprovision")) | sort_by(.timestamp) | reverse | first'

# Check AAP job for error details
AAP_JOB_ID=$(kubectl get computeinstance <name> -o json | jq -r '.status.jobs // [] | map(select(.type == "deprovision")) | sort_by(.timestamp) | reverse | first | .jobID')
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.result_traceback'
```

**Recovery:**

```bash
# 1. Manually clean up cloud resources
# - Log in to infrastructure provider (OpenShift Virtualization, VMware, etc.)
# - Verify VM/resources exist
# - Manually delete VM, networks, storage, etc.

# 2. Remove finalizers to allow CR deletion
kubectl patch computeinstance <name> -p '{"metadata":{"finalizers":[]}}' --type=merge

# 3. Delete CR
kubectl delete computeinstance <name>

# Verify deletion
kubectl get computeinstance <name>
# Expected: NotFound
```

**Prevention:**
- Ensure deprovision playbooks have proper error handling
- Test deprovision playbooks in non-production environments
- Monitor deprovision job failures and fix underlying issues

### Duplicate Job Triggers

**Symptoms:**
- Multiple AAP jobs triggered for same ComputeInstance
- Different job IDs in CR status on each reconciliation

**Diagnosis:**

```bash
# Check CR jobs array history
kubectl get computeinstance <name> -o json | jq '.status.jobs'

# List AAP jobs for this resource
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/?name__contains=<name>" | jq '.results[] | {id, status, created}'
```

**Root Cause:**
- Status update failed after job trigger (conflict, timeout)
- Job ID not persisted before controller restart

**Recovery:**

```bash
# 1. Identify the correct/latest job
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/?name__contains=<name>&order_by=-id" | jq '.results[0]'

# 2. Cancel duplicate jobs if needed
for job_id in <duplicate-job-ids>; do
  curl -X POST -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$job_id/cancel/"
done

# 3. Update CR status with correct job ID (if needed)
kubectl edit computeinstance <name>
# Manually update status.jobs array to remove duplicate job entries or correct jobID values
```

**Prevention:**
- Operator already uses `retry.RetryOnConflict` for status updates
- Ensure sufficient controller resources (CPU, memory) to avoid timeouts
- Increase API server timeout if needed

### Controller Crash During Job Execution

**Symptoms:**
- Controller pod restarted during provisioning
- Job ID exists in CR status
- Uncertain if job completed

**Diagnosis:**

```bash
# Check latest provision job ID in CR status
AAP_JOB_ID=$(kubectl get computeinstance <name> -o json | jq -r '.status.jobs // [] | map(select(.type == "provision")) | sort_by(.timestamp) | reverse | first | .jobID')

# Check job status in AAP
curl -H "Authorization: Bearer $AAP_TOKEN" "$AAP_URL/jobs/$AAP_JOB_ID/" | jq '.status'
```

**Recovery:**

AAP Direct Provider is designed for crash recovery:

```bash
# 1. Controller will automatically resume on next reconciliation
# 2. Job ID is persisted in CR status
# 3. Controller will poll job status from AAP
# 4. No manual intervention needed

# Verify recovery:
kubectl logs -n cloudkit-system deployment/cloudkit-operator-controller-manager -f | grep <name>
# Should show: "checking provision job status" and resume from there
```

**Expected Behavior:**
- Controller resumes polling job status
- No duplicate jobs are triggered
- Job completion is detected and CR status updated
