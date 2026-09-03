# BCM Backend Guide

Admin guide for configuring, operating, and troubleshooting the NVIDIA Base
Command Manager (BCM) inventory backend of the bare-metal-fulfillment-operator.
Companion to the [Metal3 backend guide](metal3-backend.md).

> Examples use `oc`; `kubectl` works identically on OpenShift.

## Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: LiteNode requirements (Day-0)](#step-1-litenode-requirements-day-0)
- [Step 2: Configure and deploy the operator (Helm)](#step-2-configure-and-deploy-the-operator-helm)
- [What the chart generates](#what-the-chart-generates)
- [RBAC](#rbac)
- [Step 3: Verify](#step-3-verify)
- [Scoped BCM certificate profile](#scoped-bcm-certificate-profile)
- [Reference: Helm `bcm.*` values](#reference-helm-bcm-values)
- [Troubleshooting](#troubleshooting)
  - [Common issues](#common-issues)
  - [Diagnostics](#diagnostics)
  - [Recovery procedures](#recovery-procedures)
  - [Backend switching](#backend-switching)

## Overview

The BCM backend lets OSAC provision bare-metal hosts from a pool of servers
managed by [NVIDIA Base Command Manager](https://docs.nvidia.com/base-command-manager/index.html)
(GPU clusters, DGX systems, and multi-vendor hardware). It is an **inventory**
backend for the bare-metal-fulfillment-operator, alongside the OpenStack and
Metal3 inventory backends.

**Hybrid design — BCM for inventory, Metal3 for management.** BCM is the source of
truth for *which hosts exist and which are free*; it does not perform power
management or OS provisioning here. When a tenant requests a `BareMetalInstance`,
the operator:

1. asks BCM for a free host (`FindFreeHost`), filtering by `resource_class`;
2. reserves it in BCM by writing an opaque `osac_instance_id` into the device's
   `extra_values` (`AssignHost`);
3. creates a Metal3 `BareMetalHost` (BMH) CR pointing at that host's BMC, so real
   Metal3/Ironic inspects and provisions it;
4. waits for the BMH to become `available` (`IsBMHReady`) before marking the
   instance ready and handing off to the management (power) path.

On deletion the operator clears `osac_instance_id` in BCM and deletes the
on-demand BMH (`UnassignHost`), returning the host to the free pool.

Key properties:

- **On-demand BMH CRs.** Unlike the Metal3 inventory backend (which selects from
  pre-existing BMHs), the BCM backend *creates and deletes* BMH CRs — hence the
  extra RBAC (see [RBAC](#rbac)).
- **BMC credentials come from BCM `bmcSettings`** (device → category → partition
  inheritance); the operator creates the Metal3 BMC credential Secret itself.
- **Tenant isolation.** Only the opaque `osac_instance_id` (the BareMetalInstance
  UID) is written to BCM — no tenant name, namespace, or org.
- **Idempotent + crash-safe.** The BCM write precedes BMH creation; every step is
  idempotent and BMH names are deterministic (the BCM hostname).
- **Minimum BCM version 10.25.3**, enforced at startup via `GET /rest/v1/version`.

## Prerequisites

- A running **BCM head node, version 10.25.3 or later**. Its JSON API must be
  reachable from the cluster (typically `https://<bcm-head>:8081`, `/json`, mTLS).
- An **mTLS client certificate + key (PEM)** for BCM's JSON API. On a BCM head
  node these are `~/.cm/admin.pem` / `~/.cm/admin.key`. See
  [Scoped profile](#scoped-bcm-certificate-profile) for least-privilege.
- **Metal3 / Ironic** deployed in the cluster — the BCM backend provisions through
  Metal3; it does not do power/OS management itself.
- The bare-metal-fulfillment-operator Helm chart (directly, or via `osac-installer`).

## Step 1: LiteNode requirements (Day-0)

The bare-metal hosts must already be registered in BCM as **LiteNodes** before
OSAC can allocate them (a BCM administration task, using BCM's own tooling). Each
LiteNode the operator should be able to use must carry:

| Field | Requirement |
|-------|-------------|
| `hostname` | Unique, DNS-safe (`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, ≤63 chars, no `/`). Becomes the BMH CR name; non-conforming hosts are skipped. |
| `mac` | Boot NIC MAC (`aa:bb:cc:dd:ee:ff`, non-zero). Becomes the BMH `bootMACAddress` (how Metal3/Ironic identifies the boot NIC) and is used for Redfish BMC discovery. Hosts with a missing/invalid MAC are skipped. |
| `extra_values.resource_class` | **The operator matches requests on this** — it becomes the OSAC *host type* (the `host_type` metric label). A LiteNode without `resource_class` in its `extra_values` is skipped by `FindFreeHost`. |
| `bmcSettings` | BMC credentials — see below. |

> Set `resource_class` in the device's `extra_values` with `cmdevice.updateDevice`
> (GET the device, add the value, PUT the whole object back — BCM does whole-object
> replacement).

### BMC credentials

The operator sources BMC credentials from BCM **`bmcSettings`** and creates the
Metal3 BMC credential Secret itself (operator-managed, label-guarded — it never
touches admin-provided Secrets).

`bmcSettings` may be set at the **device**, **category**, or **partition** level;
the operator resolves them in that order (device → category → partition), so
credentials can be set once for a whole category or partition. If none is found at
any level, assignment fails with `no BMC credentials configured in BCM for host
"<name>" — populate bmcSettings on the device, its category, or its partition`.

### BMC address

The operator resolves the host's BMC address in priority order:
1. `extra_values.osac_bmc_address` if present (e.g. `ipmi://10.0.0.1` or a
   `redfish-virtualmedia+https://…` URL) — set this to skip discovery;
2. otherwise, Redfish discovery from the LiteNode's `NetworkBmcInterface`.

## Step 2: Configure and deploy the operator (Helm)

The operator chart is **values-driven**: you set `bcm.*` values and the chart
**generates** the inventory, management, and mTLS Secrets. Do **not** hand-create
those Secrets — the chart owns them and fails the render if required values are
missing.

```yaml
# values.yaml for the bare-metal-fulfillment-operator chart
bcm:
  enabled: true                         # turns on the BCM backend + Secret generation
  url: "https://bcm-head:8081"          # required
  cert: |                               # required — raw PEM mTLS client cert (from ~/.cm/admin.pem)
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  key: |                                # required — raw PEM client key (from ~/.cm/admin.key)
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
  caCert: ""                            # optional — CA to verify the BCM server cert; empty = system trust
  insecureSkipVerify: false             # true only in test environments
  hostClass: "bcm"                      # host class stamped on assigned hosts
  bmhNamespace: "osac-baremetal"        # required — namespace where the operator creates BMH CRs

# secrets:
#   bcmCerts: "osac-bcm-certs"          # OPTIONAL — only to rename the generated Secret (default: osac-bcm-certs)
```

Only four values are mandatory: **`bcm.url`, `bcm.cert`, `bcm.key`, and
`bcm.bmhNamespace`**. If `bcm.enabled: true` but any is unset, the Helm render
fails with an explicit message. Everything else has a working default — including
`secrets.bcmCerts` (`osac-bcm-certs`); override it only to rename the generated
Secret (the chart renames the Secret, its mount, and the inventory
`credentialsSecret` together, so it stays consistent).

Deploy/upgrade with these values (via `osac-installer` or the operator chart).

> **Certificate rotation:** replace `bcm.cert`/`bcm.key` (upgrade the release, or
> edit the generated Secret) — the operator's `certwatcher` reloads the cert with
> **no restart**.

## What the chart generates

When `bcm.enabled: true`, the chart creates three Secrets in the release namespace:

| Secret (default name) | Rendered content |
|-----------------------|------------------|
| `osac-bcm-certs` (`secrets.bcmCerts`) | `stringData.tls.crt` = `bcm.cert`, `tls.key` = `bcm.key`, `ca.crt` = `bcm.caCert` (if set). Mounted at `/etc/osac/certs/<secrets.bcmCerts>/` (default `/etc/osac/certs/osac-bcm-certs/`). |
| `osac-inventory-config` | `inventory.yaml`: `type: bcm`, `hostClass: <bcm.hostClass>`, `options.bcm.{url, credentialsSecret: <secrets.bcmCerts>, insecureSkipVerify}`. |
| `osac-management-config` | `management.yaml`: `type: metal3`, `options.metal3.namespace: <bcm.bmhNamespace>`. |

BCM inventory **requires** Metal3 management — the operator errors out otherwise.
The BMH namespace is taken from `management.yaml` and reused by the BCM client, so
it's configured in one place (`bcm.bmhNamespace`).

## RBAC

The BCM backend needs more than the other inventory backends because it *creates*
and *deletes* BMH CRs and manages the Metal3 BMC credential Secret. The chart
ships:

- `metal3.io/baremetalhosts`: `create`, `delete` (plus the existing
  `get/list/watch/update/patch`) — in the operator **ClusterRole**, always present.
- **Secrets** in the BMH namespace (`get`/`create`/`update`/`delete`) via a
  **namespace-scoped `Role`+`RoleBinding`**, generated only when `bcm.enabled` and
  `bcm.bmhNamespace` are set. The operator reads Secrets through an uncached client
  (no cluster-wide list/watch).

## Step 3: Verify

```bash
# 1. Operator started cleanly with the BCM backend (version check + BMH manager)
oc logs deploy/bare-metal-fulfillment-operator -n osac | grep -iE "bcm|version|BMH manager"

# 2. The generated Secrets exist (the cert Secret name follows secrets.bcmCerts)
oc get secret osac-inventory-config osac-management-config "${BCM_CERTS_SECRET:-osac-bcm-certs}" -n osac

# 3. A registered LiteNode carries extra_values.resource_class.
#    Verify the BCM server cert with --cacert (the CA you configured as bcm.caCert);
#    only fall back to -k for a self-signed cert in a throwaway test environment.
curl --cacert bcm-ca.pem --cert admin.pem --key admin.key -X POST https://<bcm-head>:8081/json \
  -H 'Content-Type: application/json' \
  -d '{"service":"cmdevice","call":"getDevice","args":["<hostname>"]}' \
  | jq '.extra_values'

# 4. End-to-end: create a BareMetalInstance (fulfillment API / osac CLI), watch it
#    reach Ready, and confirm the operator created a BMH (use your bcm.bmhNamespace):
oc get baremetalhosts -n <bcm.bmhNamespace>   # e.g. osac-baremetal
```

`osac_bcm_hosts_available{host_type=...}` non-zero for your host types confirms the
pool is visible — see [Diagnostics](#diagnostics) for reaching the metrics endpoint.

## Scoped BCM certificate profile

The operator calls exactly these BCM methods: `cmdevice.getDevices`,
`cmdevice.getDevice`, `cmdevice.updateDevice`, `cmdevice.getCategories`,
`cmpart.getPartitions`, and `GET /rest/v1/version`. Production deployments SHOULD
use a BCM certificate **profile scoped to these methods** rather than the full
`admin` profile — a BCM-side step, no OSAC change required.

> The `getCategories`/`getPartitions` calls are required for BMC-credential
> inheritance; a profile that omits them breaks credential resolution for LiteNodes
> that inherit `bmcSettings` from a category or partition.

## Reference: Helm `bcm.*` values

| Value | Required | Meaning |
|-------|----------|---------|
| `bcm.enabled` | yes | Enables the backend and generates the Secrets + RBAC. |
| `bcm.url` | yes | BCM JSON API base URL (e.g. `https://bcm-head:8081`). |
| `bcm.cert` | yes | Raw PEM mTLS client certificate. |
| `bcm.key` | yes | Raw PEM mTLS client key. |
| `bcm.bmhNamespace` | yes | Namespace where the operator creates BMH CRs (also the Metal3 management namespace). |
| `bcm.caCert` | no | Raw PEM CA to verify the BCM server cert (empty → system trust store). |
| `bcm.insecureSkipVerify` | no | Disable BCM server-cert verification — **test only** (default `false`). |
| `bcm.hostClass` | no | Host class stamped on assigned hosts (default `bcm`). |
| `secrets.bcmCerts` | no | Name of the generated mTLS Secret (default `osac-bcm-certs`). |

## Troubleshooting

> **Where BCM problems surface:** most BCM connectivity/assignment/readiness
> failures are **not** written to `BareMetalInstance.status.conditions` — the
> instance simply stays in `Allocating` while the operator retries. The detail is
> in the **operator logs** and the **metrics**. Only three allocation condition
> messages appear in status: `Allocating BareMetalInstance`, `No matching hosts
> available`, and `BareMetalInstance allocated a host (<name>) from <backend>`.

### Common issues

| Issue | Symptoms | Root cause | Resolution |
|-------|----------|------------|------------|
| Operator won't start with BCM | Startup error mentioning version | BCM older than **10.25.3** (enforced via `GET /rest/v1/version`) | Upgrade BCM to ≥ 10.25.3. |
| BCM unreachable | Instances stuck in `Allocating`; operator logs `Failed to find a free host` / `Failed to assign host`; `osac_bcm_api_requests_total{status="error"}` climbing | Network/DNS/firewall to `bcm.url`, or BCM down | Test reachability: `curl --cacert bcm-ca.pem --cert admin.pem --key admin.key https://<bcm-head>:8081/rest/v1/version` (client cert needed — `/json` requires mTLS). Fix routing/DNS. Retries are automatic. |
| mTLS certificate expired/invalid | All BCM calls fail with a TLS error; all BCM-backed instances stall | Client cert expired, or `ca.crt` can't verify the BCM server | Update `bcm.cert`/`bcm.key` (Helm upgrade, or edit the `osac-bcm-certs` Secret). `certwatcher` reloads — **no restart**. Self-signed BCM server cert in test → `bcm.insecureSkipVerify: true`. |
| Auth/permission error from BCM | TLS handshake failure or "does not allow access"; `status="error"` on BCM calls | Client cert not accepted, or its profile is too narrow | Confirm the cert's profile allows all six operator calls (see [scoped profile](#scoped-bcm-certificate-profile)). The `admin` profile always works. |
| No matching host / pool exhausted | Instance stays in `Allocating`; condition `No matching hosts available`; `osac_bcm_hosts_available{host_type="X"}` is `0` | No free LiteNode with a matching `extra_values.resource_class` | Register more LiteNodes with the needed `resource_class` **in `extra_values`** (see [Step 1](#step-1-litenode-requirements-day-0)), or free hosts by deleting instances. |
| Hosts registered but never selected | `osac_bcm_hosts_available` is `0` even though LiteNodes exist | `resource_class` is not set in the device's `extra_values` — the operator only reads `extra_values.resource_class` | Set `extra_values.resource_class` on each device (`getDevice` → add → `updateDevice`). |
| No BMC credentials | Instance stuck; operator log: `no BMC credentials configured in BCM for host "<name>" — populate bmcSettings on the device, its category, or its partition` | LiteNode (and its category/partition) have no usable `bmcSettings` | Set `bmcSettings` at the device, category, or partition level. |
| BMH stuck registering/inspecting | Instance stays in `Allocating`; BMH `provisioning.state` not `available` | Metal3/Ironic can't reach or inspect the BMC (bad address, creds, or BMC offline) | Inspect the BMH (below); verify the resolved BMC address + credentials and that the BMC is reachable/powered. |
| BMH never becomes ready | Instance stuck in `Allocating` indefinitely | Hardware/BMC problem; readiness retries forever (no timeout) | Diagnose the BMH/BMC; if unrecoverable, the tenant deletes the `BareMetalInstance`. |
| BCM device removed mid-flow | Instance stuck; operator log: `BCM device "<name>" no longer exists in BCM inventory but BareMetalHost CR exists — delete the BareMetalInstance or re-register the device in BCM` | A LiteNode was deleted in BCM after its BMH was created | Re-register the device in BCM, **or** delete the orphaned BMH and the `BareMetalInstance`. |

### Diagnostics

#### Prometheus metrics (currently emitted)

| Metric | Type | Labels | Use |
|--------|------|--------|-----|
| `osac_bcm_api_requests_total` | Counter | `method` (`getDevices`/`getDevice`/`updateDevice`/`getCategories`/`getPartitions`), `status` (`success`/`error`) | Rising `status="error"` → connectivity/auth/API problems |
| `osac_bcm_api_duration_seconds` | Histogram | `method` | BCM API latency |
| `osac_bcm_hosts_available` | Gauge | `host_type` | Free LiteNodes per host type (updated on each `FindFreeHost`). `0` → no free host for that type |

**The metrics endpoint is HTTPS on `:8443` and token-authenticated** — the chart
sets `--metrics-bind-address=:8443` with `--metrics-secure` (protected by the
`metrics-reader` role); there is no plaintext endpoint unless you override that
flag. The operator image (ubi-minimal) also has no `wget`/`curl`. So scrape via
Prometheus, or manually with a bearer token:

```bash
oc -n osac port-forward deploy/bare-metal-fulfillment-operator 8443:8443 &
TOKEN=$(oc create token <serviceaccount-bound-to-metrics-reader> -n osac)
curl -ksS -H "Authorization: Bearer ${TOKEN}" https://localhost:8443/metrics | grep osac_bcm_
```

> **Not yet emitted (planned).** A `osac_bcm_bmh_readiness_duration_seconds`
> histogram and Kubernetes events (`BCMHostAssigned`, `BCMHostReleased`,
> `BCMConnectionError`) are described in the design but **not implemented** in the
> current release — don't build alerts/runbooks on them yet.

#### Status and conditions

```bash
oc get baremetalinstance <name> -n <ns> -o jsonpath='{.status.phase}{"\n"}'
oc get baremetalinstance <name> -n <ns> -o yaml | yq '.status.conditions'
```

A BCM/BMC problem shows as a *persistent* `Allocating` — the cause is in the logs.

#### Key operator log messages

```bash
oc logs deploy/bare-metal-fulfillment-operator -n osac | grep -iE "bcm|baremetalhost|free host|assign|readiness|bmc"
```

| Log pattern | Meaning |
|-------------|---------|
| `Failed to find a free host` | `FindFreeHost` failed — usually BCM unreachable, or no matching `resource_class`. |
| `Failed to assign host` | `AssignHost`/BCM write or BMH creation failed. |
| `Host reserved but not ready yet, requeuing` (V=1) | Normal readiness wait — BMH not yet `available`. |
| `no BMC credentials configured in BCM for host "…"` | Missing `bmcSettings` at device/category/partition. |
| `BCM device "…" no longer exists in BCM inventory but BareMetalHost CR exists …` | Device removed from BCM mid-flow — orphaned BMH. |

The operator logs only `hostname`/`call`/`status`/`bmcAddress` fields — it never
logs full device objects or `bmcSettings` (which contain BMC credentials).

#### Inspect the BMH and BCM

```bash
# The BMH name equals the BCM hostname
oc get bmh <hostname> -n <bmh-namespace> -o jsonpath='{.status.provisioning.state} {.status.operationalStatus}{"\n"}'
oc get bmh <hostname> -n <bmh-namespace> -o yaml | yq '.status.errorMessage, .spec.bmc.address'

# Inspect BCM directly (verify the server cert with --cacert; -k only for self-signed test)
curl --cacert bcm-ca.pem --cert admin.pem --key admin.key -X POST https://<bcm-head>:8081/json \
  -H 'Content-Type: application/json' \
  -d '{"service":"cmdevice","call":"getDevice","args":["<hostname>"]}' | jq '.extra_values'
```

An assigned host has `extra_values.osac_instance_id` set; a free host does not.

### Recovery procedures

**BCM unreachable / cert expired.** No manual recovery for transient outages — all
BCM calls and BMH readiness retry **indefinitely**, and instances resume when BCM
returns or the cert is replaced (certwatcher, no restart).

**Orphaned on-demand BMH.** If a `BareMetalInstance` is gone but its BMH remains
(e.g. a device was removed in BCM mid-flow), confirm it's unused and delete it:

```bash
oc get bmh <hostname> -n <bmh-namespace> -o jsonpath='{.spec.consumerRef}{"\n"}'  # confirm empty
oc delete bmh <hostname> -n <bmh-namespace>
```
`UnassignHost` is idempotent — a null BCM device is treated as already cleaned up.

**Host stuck / never ready.** Readiness has no timeout — if a host never reaches
`available`, the tenant deletes the `BareMetalInstance`.

### Backend switching

Switching the inventory backend type is **not seamless** — each backend uses
different host identifiers and cleanup logic, so a new backend cannot deprovision
instances allocated by the old one.

**Switch away from BCM — BCM reachable (the clean path):**
1. **Drain.** Delete **all** BCM-backed `BareMetalInstance`s and let each finish
   deleting — this triggers `UnassignHost` (clears BCM `osac_instance_id`, deletes
   the on-demand BMH and its operator-managed BMC Secret). Verify none remain.
2. Reconfigure the operator's inventory backend (`bcm.enabled: false` and set the
   replacement backend's values) and upgrade.
3. Restart/roll out the operator.

**Switch away from BCM — BCM unreachable (forced):** a clean drain is **not
possible** while BCM is down — deletion calls `UnassignHost`, which needs BCM, so
each `BareMetalInstance` hangs in `Deleting` (the operator does not remove its
inventory finalizer until `UnassignHost` succeeds, and BCM calls retry
indefinitely). You can still force it, but you own the cleanup:

1. Delete the `BareMetalInstance`s (they will sit in `Deleting`).
2. Force-remove the operator's finalizers so they can be garbage-collected:
   ```bash
   oc patch baremetalinstance <name> -n <ns> --type=json \
     -p='[{"op":"remove","path":"/metadata/finalizers"}]'
   ```
3. Manually delete the orphaned on-demand BMH CRs and their operator-managed BMC
   Secrets in `<bcm.bmhNamespace>`.
4. Reconfigure the backend (`bcm.enabled: false` + the replacement) and roll out
   the operator.
5. **When BCM comes back**, the affected devices still carry a stale
   `osac_instance_id` (the reservation was never cleared), so those hosts look
   assigned forever. Clear it on each: GET the device, remove
   `extra_values.osac_instance_id`, and `cmdevice.updateDevice` the whole object.

> ⚠️ Forcing removes the operator's safety net — you own the BMH, Secret, and
> BCM-reservation cleanup. Prefer the clean path whenever BCM can be brought back,
> even briefly.

**Switching *to* BCM:** existing instances from a previous backend are unaffected —
they already have `hostClass` set, so they stay on the management path; only new
instances use BCM.
