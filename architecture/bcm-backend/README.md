# BCM Inventory Backend

Architecture documentation for the NVIDIA Base Command Manager (BCM) inventory
backend in the OSAC bare-metal-fulfillment-operator.

**Guides:**
- [Configuration Guide](configuration.md) — Day-0 setup: mTLS, inventory/management
  config, LiteNode registration, Helm deployment, verification
- [Troubleshooting Guide](troubleshooting.md) — failure scenarios, diagnostics
  (metrics/logs), recovery, and backend switching

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [References](#references)

## Overview

The BCM backend lets OSAC provision bare-metal hosts from a pool of servers
managed by [NVIDIA Base Command Manager](https://docs.nvidia.com/base-command-manager/index.html)
(GPU clusters, DGX systems, and multi-vendor hardware). It is a second
**inventory** backend for the bare-metal-fulfillment-operator, alongside the
existing OpenStack and Metal3 inventory backends.

**Hybrid design — BCM for inventory, Metal3 for management.** BCM is the source of
truth for *which hosts exist and which are free*; it does not perform power
management or OS provisioning in this integration. When a tenant requests a
`BareMetalInstance`, the operator:

1. Asks BCM for a free host (`FindFreeHost`), filtering by `resource_class`.
2. Reserves it in BCM by writing an opaque `osac_instance_id` into the device's
   `extra_values` (`AssignHost`).
3. Creates a Metal3 `BareMetalHost` (BMH) CR pointing at that host's BMC, so real
   Metal3/Ironic inspects and provisions it.
4. Waits for the BMH to become `available` (`IsBMHReady`) before marking the
   instance ready and handing off to the management (power) path.

On deletion the operator clears `osac_instance_id` in BCM and deletes the
on-demand BMH (`UnassignHost`) — returning the host to the free pool.

> **Minimum BCM version: 10.25.03.** The operator enforces this at startup via
> `GET /rest/v1/version` (mTLS). Older versions are rejected.

## Architecture

### Components

| Component | Role |
|-----------|------|
| BCM JSON API (`/json`, mTLS) | Inventory source of truth. Operator uses `cmdevice.getDevices`, `getDevice`, `updateDevice`, and `cmdevice.getCategories` / `cmpart.getPartitions` (BMC-credential inheritance) |
| BCM REST API (`/rest/v1/version`) | Startup version check only |
| BCM inventory client (`internal/inventory/bcm.go`) | `FindFreeHost` / `AssignHost` / `UnassignHost`; owns `extra_values` bookkeeping |
| BMH lifecycle manager (`internal/baremetalhost/`) | Creates/deletes on-demand BMH CRs and the Metal3 BMC credential Secret; `IsBMHReady` |
| Metal3 / Ironic | Actual inspection, provisioning, and power management (unchanged) |

### Request flow (Day-2)

```text
Tenant → fulfillment-service → BareMetalInstance CR → BMI controller
  reconcileInventory (Allocating):
    FindFreeHost (BCM getDevices, filter resource_class → host_type)
    AssignHost   (BCM updateDevice: set osac_instance_id) → CreateBMH → IsBMHReady
  reconcileManagement (HostClass set): Metal3/Ironic power + AAP provisioning
```

Key properties:
- **On-demand BMH CRs.** Unlike the Metal3 inventory backend (which selects from
  pre-existing BMHs), the BCM backend *creates and deletes* BMH CRs. This requires
  `create`/`delete` on `metal3.io/baremetalhosts` and namespace-scoped Secret
  permissions (see [Configuration](configuration.md#rbac)).
- **BMC credentials come from BCM `bmcSettings`** (device → category → partition
  inheritance). The operator reads them and creates the Metal3 BMC credential
  Secret itself — admins do **not** stage per-host BMC Secrets.
- **Tenant isolation.** Only the opaque `osac_instance_id` (the BareMetalInstance
  UID) is written to BCM. No tenant name, namespace, or org is exposed.
- **Idempotent + crash-safe.** BCM write precedes BMH creation; every step is
  idempotent, and BMH names are deterministic (the BCM hostname), so a crash
  recovers cleanly on the next reconcile.

## Configuration

You configure the backend through the operator chart's **`bcm.*` Helm values**
(`bcm.enabled`, `url`, `cert`, `key`, `bmhNamespace`, …). The chart **generates**
the three Secrets that drive the backend (full walkthrough in the
[Configuration Guide](configuration.md)):

- `osac-inventory-config` → `inventory.yaml` with `type: bcm` and `options.bcm`
  (`url`, `credentialsSecret`, `insecureSkipVerify`).
- `osac-management-config` → `management.yaml` with `type: metal3` (BCM inventory
  **requires** Metal3 management).
- `osac-bcm-certs` → the mTLS client cert (`tls.crt`, `tls.key`, optional `ca.crt`),
  mounted at `/etc/osac/certs/osac-bcm-certs/`.

## References

- [NVIDIA Base Command Manager docs](https://docs.nvidia.com/base-command-manager/index.html)
- [BCM 11 Developer Manual (REST + JSON API)](https://docs.nvidia.com/base-command-manager/manuals/11/developer-manual.pdf)
- OSAC feature: OSAC-1339 · Design: `enhancement-proposals/enhancements/OSAC-1339-bcm-backend/design.md`
- Operator code: `bare-metal-fulfillment-operator/internal/{bcmclient,inventory,baremetalhost}/`
- For general OSAC architecture, see the [architecture README](../README.md).
