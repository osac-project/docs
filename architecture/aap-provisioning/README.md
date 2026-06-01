# AAP Provisioning

Architecture documentation for AAP-based provisioning in the OSAC operator.

**Operational Guides:**
- [Operations Guide](operations.md) - Deployment and monitoring
- [Testing Guide](testing.md) - Manual testing, unit tests, integration tests
- [Troubleshooting Guide](troubleshooting.md) - Common issues, debugging, recovery procedures

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [References](#references)

## Overview

The OSAC operator orchestrates infrastructure provisioning through Ansible Automation Platform (AAP). The operator launches AAP job or workflow templates over the REST API and polls job status until operations complete.

**Problem Statement:** Earlier webhook-based approaches had limitations:
- No job status feedback (fire-and-forget)
- Unreliable completion signals
- No visibility into failures or progress
- Orphaned cloud resources possible

**Solution:** Direct AAP REST API integration with full job lifecycle management:
- Real-time job status from the AAP API
- Job cancellation during deletion
- Operator-managed finalizers
- Crash recovery via persisted job IDs in CR status
- Full error tracebacks from failed jobs

For architectural context, see the [main OSAC architecture documentation](../README.md).

## Architecture

### Provisioning Flow

**Core Capabilities:**
- Trigger provisioning/deprovisioning operations via AAP templates
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

![AAP Provisioning Component Diagram](images/AAP%20Direct%20Model%20-%20Component%20Diagram.png)

### AAP Integration (REST API)

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

**Template Auto-Detection:**

The provider automatically determines whether templates are `job_template` or `workflow_job_template` by querying the AAP API. Results are cached in memory.

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

**Asynchronous Deletion:** Kubernetes sets `deletionTimestamp` immediately. The operator handles cleanup asynchronously through reconciliation loops, waiting for provision completion before triggering deprovision.

![AAP Direct Deletion](images/AAP%20Direct%20Model%20-%20Deletion%20During%20Provisioning.png)

### Implementation

**Code Organization:**

```
osac-operator/
├── pkg/provisioning/
│   ├── provider.go           # Interface definitions
│   └── aap_provider.go       # AAP implementation
├── internal/aap/
│   ├── client.go             # AAP REST client
│   └── types.go              # API types
└── internal/controller/
    └── computeinstance_controller.go
```

**Key Patterns:**

**Preventing Parallel Operations:**
- Operator prevents provision and deprovision from running simultaneously
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

### Required Settings

| Setting | Description |
|---------|-------------|
| `OSAC_AAP_URL` | AAP Controller API URL (with `/api/controller`) |
| `OSAC_AAP_TOKEN` | Secret reference to AAP OAuth2 token |
| `OSAC_AAP_TEMPLATE_PREFIX` | Prefix for AAP template names (e.g., `osac`) |

### Optional Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `OSAC_AAP_STATUS_POLL_INTERVAL` | Job status poll interval | `30s` |
| `OSAC_AAP_INSECURE_SKIP_VERIFY` | Skip TLS verification for AAP | `false` |
| `OSAC_MAX_JOB_HISTORY` | Max job entries retained in CR status | `10` |

### Example Configuration

```yaml
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

**AAP Token Secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: osac-aap-api-token
  namespace: osac-operator-system
type: Opaque
stringData:
  token: "your-aap-oauth2-token"
```

**Requirements:**
- AAP Controller accessible from operator
- OAuth2 token with launch and read permissions
- Job/workflow templates configured with `OSAC_AAP_TEMPLATE_PREFIX`
- Templates accept resource JSON in `ansible_eda.event.payload` format
- Templates should NOT manage finalizers

### Configuration Notes

- URL must include `/api/controller` suffix
- Template names are resolved from prefix + resource type (e.g., `osac-create-compute-instance`)
- Poll interval: 30s–60s recommended (balance responsiveness vs API load)
- Templates can be `job_template` or `workflow_job_template`

## References

### Documentation

- **[Operations Guide](operations.md)** - Deployment and monitoring
- **[Testing Guide](testing.md)** - Manual testing scenarios, unit tests, integration tests
- **[Troubleshooting Guide](troubleshooting.md)** - Common issues, debugging commands, recovery procedures
- [Main OSAC Architecture](../README.md) - Overall architecture
- [Enhancement Proposal](https://github.com/osac-project/enhancement-proposals/tree/main/enhancements/aap-provisioning-abstraction) - Original design

### Code

- [osac-operator](https://github.com/osac-project/osac-operator) - Controller implementation
- `pkg/provisioning/` - Provider interface and AAP implementation
- `internal/aap/` - AAP REST client

### External

- [AAP API Documentation](https://docs.ansible.com/automation-controller/latest/html/controllerapi/index.html)
- [AAP Job Launch API](https://docs.ansible.com/automation-controller/latest/html/controllerapi/api_ref.html#/Jobs)
- [AAP Authentication](https://docs.ansible.com/automation-controller/latest/html/userguide/applications_auth.html)
