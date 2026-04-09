# AAP Provisioning Abstraction and Direct Integration

Architecture documentation for the AAP provisioning system.

**Operational Guides:**
- [Operations Guide](operations.md) - Deployment, migration, monitoring
- [Testing Guide](testing.md) - Manual testing, unit tests, integration tests
- [Troubleshooting Guide](troubleshooting.md) - Common issues, debugging, recovery procedures

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [References](#references)

## Overview

The OSAC Controller orchestrates infrastructure provisioning through Ansible Automation Platform (AAP). The controller implements a provider abstraction layer enabling flexible integration models.

**Problem Statement:** The original Event-Driven Ansible (EDA) webhook approach had limitations:
- No job status feedback (fire-and-forget)
- Unreliable completion signals (annotation-based)
- No visibility into failures or progress
- Orphaned cloud resources possible

**Solution:** The `ProvisioningProvider` interface supports two implementations:
1. **EDA Provider** - Legacy webhook-based (backward compatibility)
2. **AAP Direct Provider** - REST API-based with full job lifecycle management

**Provider Comparison:**

| Feature | EDA Provider | AAP Direct Provider |
|---------|-------------|---------------------|
| Communication | Webhook (fire-and-forget) | REST API (polling) |
| Job Status | Unknown | Real-time from AAP API |
| Job Cancellation | Not supported | Supported |
| Finalizer Management | AAP playbook | Operator |
| Crash Recovery | Limited (annotation-based) | Full (job ID in CR) |
| Error Visibility | None | Full traceback |
| Orphaned Resources | Possible | Prevented via BlockDeletionOnFailure |

For architectural context, see the [main OSAC architecture documentation](../README.md).

## Architecture

### Provider Abstraction Layer

The provisioning system is built around a provider interface that abstracts infrastructure automation triggering and job status retrieval.

**Core Capabilities:**
- Trigger provisioning/deprovisioning operations
- Query job status for running operations
- Track job lifecycle (creation → completion)
- Return job identifiers for status polling

**Job Lifecycle States:**

- **Pending** - Job created, not started
- **Waiting** - Waiting for resources/dependencies
- **Running** - Currently executing
- **Succeeded** - Completed successfully (terminal)
- **Failed** - Job failed (terminal)
- **Canceled** - Canceled before completion (terminal)
- **Unknown** - Status not recognized (non-terminal)

Terminal states indicate completion. Non-terminal states require continued polling.

**Benefits:**
- Swap backends without code changes
- Mock providers for testing
- Support multiple operational models
- Migrate providers without breaking deployments

![AAP Provisioning Component Diagram](images/AAP%20Direct%20Model%20-%20Component%20Diagram.png)

### EDA Provider (Webhook)

Legacy webhook-based provisioning using Event-Driven Ansible.

**Flow:**
1. Operator → Webhook → EDA service
2. EDA → AAP job template
3. AAP playbook provisions infrastructure
4. Playbook adds finalizer to CR
5. Playbook updates `reconciled-config-version` annotation
6. Operator detects annotation → marks CR Ready

**Characteristics:**
- Fire-and-forget (no job IDs returned)
- Passive feedback (playbook updates CR)
- Playbook manages finalizers
- Always returns `JobStateUnknown`
- Completion via annotation watch

**When to Use:**
- Legacy EDA infrastructure exists
- Simpler setup (no token management)
- Backward compatibility required

**Limitations:**
- No job failure visibility
- No progress updates
- Can't cancel jobs
- Orphaned resources possible

![EDA Deletion Flow](images/EDA%20Model%20-%20Deletion%20During%20Provisioning.png)

### AAP Direct Provider (REST API)

Direct AAP REST API integration with full job lifecycle visibility.

**Flow:**
1. Operator → AAP API launch job
2. AAP returns job ID → stored in CR status
3. Operator polls AAP for status (configurable interval)
4. Job completes → operator updates CR
5. Operator manages finalizers

**Characteristics:**
- Direct REST API calls
- Real job tracking (AAP job IDs)
- Active polling (configurable intervals)
- Full lifecycle management
- Operator-managed finalizers
- Job cancellation supported

**Advantages:**
- Real-time job status
- Full error tracebacks
- Orphaned resource prevention
- Crash recovery (persisted job IDs)
- Progress tracking capability

**Template Auto-Detection:**

The provider automatically determines whether templates are `job_template` or `workflow_job_template` by querying AAP API. Results are cached in memory.

![AAP Direct Creation](images/AAP%20Direct%20Model%20-%20Creation%20and%20Provisioning.png)

#### Deletion During Provisioning

Critical feature: Safely handle deletion while provisioning is in progress.

**Flow:**
1. User deletes ComputeInstance during provision
2. Operator checks provision job state
3. If non-terminal (Pending/Waiting/Running):
   - Cancel job via AAP API
   - Set deprovision action: `DeprovisionWaiting`
   - Requeue with backoff
4. Next reconciliation checks if provision terminal
5. Once terminal, trigger deprovision

**Asynchronous Deletion:** The deletion is non-blocking. Kubernetes sets `deletionTimestamp` immediately. The operator handles cleanup asynchronously through reconciliation loops, waiting for provision completion before triggering deprovision.

This prevents parallel provision/deprovision execution which could leave infrastructure inconsistent.

![AAP Direct Deletion](images/AAP%20Direct%20Model%20-%20Deletion%20During%20Provisioning.png)

### Implementation

**Code Organization:**

```
cloudkit-operator/
├── internal/
│   ├── provisioning/
│   │   ├── provider.go           # Interface definitions
│   │   ├── eda_provider.go       # EDA implementation
│   │   └── aap_provider.go       # AAP Direct implementation
│   ├── aap/
│   │   ├── client.go             # AAP REST client
│   │   └── types.go              # API types
│   └── controller/
│       └── computeinstance_controller.go
```

**Key Patterns:**

**Preventing Parallel Operations:**
- AAP Direct prevents provision and deprovision from running simultaneously
- Checks provision job state before deprovisioning
- Cancels running provision before starting deprovision

**Job Tracking:**
- Job info persisted in CR `status.jobs` array
- Enables crash recovery and observability
- History configurable via `OSAC_MAX_JOB_HISTORY` (default: 10)

**Job Entry Fields:**
- `jobID` - Job identifier
- `type` - `provision` or `deprovision`
- `timestamp` - Creation time (RFC3339, UTC)
- `state` - Current state
- `message` - Status message
- `blockDeletionOnFailure` - Prevent CR deletion on failure

**Idempotency:**
- Check for existing job ID in CR status
- If exists, query status instead of triggering new job
- If missing, trigger and persist immediately

## Configuration

### Provider Selection

Set via `OSAC_PROVISIONING_PROVIDER` environment variable.

**Values:**
- `aap` - AAP Direct REST API provider (default)
- `eda` - EDA webhook provider (legacy, requires explicit opt-in)

### Configuration Comparison

| Setting | EDA Provider | AAP Direct Provider |
|---------|-------------|---------------------|
| **Required** | | |
| `OSAC_PROVISIONING_PROVIDER` | `"eda"` (must set explicitly) | `"aap"` (default, optional) |
| `OSAC_COMPUTE_INSTANCE_PROVISION_WEBHOOK` | URL to EDA create endpoint | N/A |
| `OSAC_COMPUTE_INSTANCE_DEPROVISION_WEBHOOK` | URL to EDA delete endpoint | N/A |
| `OSAC_AAP_URL` | N/A | AAP Controller API URL (with `/api/controller`) |
| `OSAC_AAP_TOKEN` | N/A | Secret reference to AAP OAuth2 token |
| `OSAC_AAP_PROVISION_TEMPLATE` | N/A | AAP template name for provision |
| `OSAC_AAP_DEPROVISION_TEMPLATE` | N/A | AAP template name for deprovision |
| **Optional** | | |
| `OSAC_MINIMUM_REQUEST_INTERVAL` | Min time between webhook calls (e.g., `10m`) | N/A |
| `OSAC_AAP_STATUS_POLL_INTERVAL` | N/A | Job status poll interval (default: `30s`) |

### EDA Provider Example (Legacy)

```yaml
env:
  - name: OSAC_PROVISIONING_PROVIDER
    value: "eda"  # Must be set explicitly; AAP direct is the default
  - name: OSAC_COMPUTE_INSTANCE_PROVISION_WEBHOOK
    value: "http://innabox-eda-service:5000/create-compute-instance"
  - name: OSAC_COMPUTE_INSTANCE_DEPROVISION_WEBHOOK
    value: "http://innabox-eda-service:5000/delete-compute-instance"
  - name: OSAC_MINIMUM_REQUEST_INTERVAL
    value: "10m"
```

**Requirements:**
- EDA service deployed and accessible
- EDA webhooks configured
- AAP playbooks handle finalizers
- Playbooks update reconciled-config-version annotation

### AAP Direct Provider Example

```yaml
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

**AAP Token Secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aap-credentials
  namespace: cloudkit-system
type: Opaque
stringData:
  token: "your-aap-oauth2-token"
```

**Token Creation:**

Via AAP UI: Access → Users → [user] → Tokens → Add (Scope: Write)

Via AAP API:
```bash
curl -X POST https://aap-url/api/v2/tokens/ \
  -u username:password \
  -H "Content-Type: application/json" \
  -d '{
    "description": "cloudkit-operator token",
    "application": null,
    "scope": "write"
  }'
```

**Token Permissions:**
- Launch configured templates
- Query job status
- Cancel jobs

**Requirements:**
- AAP Controller accessible from operator
- OAuth2 token with permissions
- Job/workflow templates configured
- Templates accept EDA event format (compatibility)
- Templates should NOT manage finalizers

### Configuration Notes

**EDA:**
- `OSAC_MINIMUM_REQUEST_INTERVAL` is client-side rate limiting, not polling
- Prevents webhook flooding during reconciliation loops
- WebhookClient caches recent requests and skips duplicates within interval

**AAP Direct:**
- URL must include `/api/controller` suffix
- Template names auto-detected (job vs workflow)
- Poll interval: 30s-60s recommended (balance responsiveness vs API load)
- Templates can be job_template or workflow_job_template

## References

### Documentation

- **[Operations Guide](operations.md)** - Deployment, migration, monitoring
- **[Testing Guide](testing.md)** - Manual testing scenarios, unit tests, integration tests
- **[Troubleshooting Guide](troubleshooting.md)** - Common issues, debugging commands, recovery procedures
- [Main OSAC Architecture](../README.md) - Overall architecture
- [Enhancement Proposal](https://github.com/osac-project/enhancement-proposals/tree/main/enhancements/aap-provisioning-abstraction) - Original design

### Code

- [cloudkit-operator](https://github.com/innabox/cloudkit-operator) - Controller implementation
- `internal/provisioning/` - Provider interface and implementations
- `internal/aap/` - AAP REST client
- `internal/controller/computeinstance_controller.go` - Controller integration

### External

- [AAP API Documentation](https://docs.ansible.com/automation-controller/latest/html/controllerapi/index.html)
- [AAP Job Launch API](https://docs.ansible.com/automation-controller/latest/html/controllerapi/api_ref.html#/Jobs)
- [AAP Authentication](https://docs.ansible.com/automation-controller/latest/html/userguide/applications_auth.html)
- [Event-Driven Ansible](https://www.ansible.com/products/event-driven-ansible)

---

**Note:** This documentation describes cloudkit-operator. The operator will be renamed to osac-operator as part of OSAC project consolidation.
