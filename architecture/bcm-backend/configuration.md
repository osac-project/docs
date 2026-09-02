# BCM Backend Configuration Guide

Step-by-step Day-0 setup for the NVIDIA Base Command Manager (BCM) inventory
backend. Audience: **Cloud Infrastructure Admin**.

For architecture, see [README.md](README.md). For diagnosing problems, see the
[Troubleshooting Guide](troubleshooting.md).

> Examples use `oc`; `kubectl` works identically on OpenShift.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: LiteNode requirements (Day-0)](#step-1-litenode-requirements-day-0)
- [Step 2: Configure and deploy the operator (Helm)](#step-2-configure-and-deploy-the-operator-helm)
- [What the chart generates](#what-the-chart-generates)
- [RBAC](#rbac)
- [Step 3: Verify](#step-3-verify)
- [Scoped BCM certificate profile](#scoped-bcm-certificate-profile)
- [Reference: Helm `bcm.*` values](#reference-helm-bcm-values)

## Prerequisites

- A running **BCM head node, version 10.25.3 or later** (the operator rejects
  older versions at startup via `GET /rest/v1/version`). The BCM JSON API must be
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
| `bmcSettings` | BMC credentials — see [BMC credentials](#bmc-credentials). |

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
**generates** the inventory, management, and mTLS Secrets for you. Do **not**
hand-create those Secrets — the chart owns them and will fail the render if the
required values are missing.

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
`bcm.bmhNamespace`**. If `bcm.enabled: true` but any of those is unset, the Helm
render fails with an explicit message. Everything else has a working default —
including `secrets.bcmCerts` (`osac-bcm-certs`); override it only if you want a
different Secret name (the chart renames the generated Secret, its mount, and the
inventory `credentialsSecret` together, so it stays consistent).

Deploy/upgrade the operator with these values (via `osac-installer` or the
operator chart directly).

> **Certificate rotation:** replace `bcm.cert`/`bcm.key` (upgrade the release, or
> edit the generated Secret) — the operator's `certwatcher` reloads the cert with
> **no restart**.

## What the chart generates

When `bcm.enabled: true`, the chart creates three Secrets in the release namespace
(useful to know for verification and troubleshooting):

| Secret (default name) | Rendered content |
|-----------------------|------------------|
| `osac-bcm-certs` (`secrets.bcmCerts`) | `stringData.tls.crt` = `bcm.cert`, `tls.key` = `bcm.key`, `ca.crt` = `bcm.caCert` (if set). Mounted at `/etc/osac/certs/osac-bcm-certs/`. |
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
  **namespace-scoped `Role`+`RoleBinding`**, generated only when
  `bcm.enabled` and `bcm.bmhNamespace` are set. The operator reads Secrets through
  an uncached client (no cluster-wide list/watch).

## Step 3: Verify

```bash
# 1. Operator started cleanly with the BCM backend (version check + BMH manager)
oc logs deploy/bare-metal-fulfillment-operator -n osac | grep -iE "bcm|version|BMH manager"

# 2. The generated Secrets exist
oc get secret osac-inventory-config osac-management-config osac-bcm-certs -n osac

# 3. A registered LiteNode carries extra_values.resource_class
curl -k --cert admin.pem --key admin.key -X POST https://<bcm-head>:8081/json \
  -H 'Content-Type: application/json' \
  -d '{"service":"cmdevice","call":"getDevice","args":["<hostname>"]}' \
  | jq '.extra_values'

# 4. End-to-end: create a BareMetalInstance (fulfillment API / osac CLI), watch it
#    reach Ready, and confirm the operator created a BMH:
oc get baremetalhosts -n osac-baremetal
```

Metrics (`osac_bcm_hosts_available{host_type=...}` non-zero for your host types)
confirm the pool is visible — see the Troubleshooting Guide for how to reach the
secured metrics endpoint.

## Scoped BCM certificate profile

The operator calls exactly these BCM methods: `cmdevice.getDevices`,
`cmdevice.getDevice`, `cmdevice.updateDevice`, `cmdevice.getCategories`,
`cmpart.getPartitions`, and `GET /rest/v1/version`. Production deployments SHOULD
use a BCM certificate **profile scoped to these methods** rather than the full
`admin` profile. This is a BCM-side step — no OSAC change required.

> The two `getCategories`/`getPartitions` calls are required for BMC-credential
> inheritance — a profile that omits them breaks credential resolution for
> LiteNodes that inherit `bmcSettings` from a category or partition.

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
