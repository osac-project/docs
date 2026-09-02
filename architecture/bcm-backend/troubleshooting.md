# BCM Backend Troubleshooting Guide

Symptom-based diagnosis, monitoring, and recovery for the BCM inventory backend.
Audience: **Cloud Infrastructure Admin**.

For setup, see the [Configuration Guide](configuration.md); for architecture, see
[README.md](README.md).

> Examples use `oc`; `kubectl` works identically on OpenShift.
>
> **Where BCM problems surface:** most BCM connectivity/assignment/readiness
> failures are **not** written to `BareMetalInstance.status.conditions` — the
> instance simply stays in `Allocating` while the operator retries. The detail is
> in the **operator logs** and the **metrics**. Only three allocation condition
> messages actually appear in status: `Allocating BareMetalInstance`,
> `No matching hosts available`, and `BareMetalInstance allocated a host (<name>)
> from <backend>`.

## Table of Contents

- [Common Issues](#common-issues)
- [Diagnostics](#diagnostics)
- [Recovery Procedures](#recovery-procedures)
- [Backend Switching](#backend-switching)

## Common Issues

| Issue | Symptoms | Root cause | Resolution |
|-------|----------|------------|------------|
| Operator won't start with BCM | Startup error mentioning version | BCM older than **10.25.3** (enforced via `GET /rest/v1/version`) | Upgrade BCM to ≥ 10.25.3. |
| BCM unreachable | Instances stuck in `Allocating`; operator logs `Failed to find a free host` / `Failed to assign host`; `osac_bcm_api_requests_total{status="error"}` climbing | Network/DNS/firewall to `options.bcm.url`, or BCM down | Test reachability from the cluster: `curl -k --cert admin.pem --key admin.key https://<bcm-head>:8081/rest/v1/version` (add the client cert — `/json` requires mTLS). Fix routing/DNS. Retries are automatic — instances recover when BCM returns. |
| mTLS certificate expired/invalid | All BCM calls fail with a TLS error; all BCM-backed instances stall | Client cert expired, or `ca.crt` can't verify the BCM server | Update `bcm.cert`/`bcm.key` (Helm upgrade, or edit the generated `osac-bcm-certs` Secret). `certwatcher` reloads — **no restart**. For a self-signed BCM server cert in test, `bcm.insecureSkipVerify: true`. |
| Auth/permission error from BCM | TLS handshake failure or "does not allow access"; `status="error"` on BCM calls | Client cert not accepted by BCM, or the cert's profile is too narrow | Confirm the cert maps to a profile allowing all six operator calls (see [scoped profile](configuration.md#scoped-bcm-certificate-profile)). The `admin` profile always works. |
| No matching host / pool exhausted | Instance stays in `Allocating`; condition `No matching hosts available`; `osac_bcm_hosts_available{host_type="X"}` is `0` | No free LiteNode with a matching `extra_values.resource_class` | Register more LiteNodes with the needed `resource_class` **in `extra_values`** (see [LiteNode requirements](configuration.md#step-1-litenode-requirements-day-0)), or free hosts by deleting instances. |
| Hosts registered but never selected | `osac_bcm_hosts_available` is `0` even though LiteNodes exist | `resource_class` is not set in the device's `extra_values` — the operator only reads `extra_values.resource_class` | Set `extra_values.resource_class` on each device (`getDevice` → add → `updateDevice`). |
| No BMC credentials | Instance stuck; operator log: `no BMC credentials configured in BCM for host "<name>" — populate bmcSettings on the device, its category, or its partition` | LiteNode (and its category/partition) have no usable `bmcSettings` | Set `bmcSettings` at the device, category, or partition level. |
| BMH stuck registering/inspecting | Instance stays in `Allocating`; BMH `provisioning.state` not `available` | Metal3/Ironic can't reach or inspect the BMC (bad address, creds, or BMC offline) | Inspect the BMH (below); verify the resolved BMC address + credentials and that the BMC is reachable/powered. |
| BMH never becomes ready | Instance stuck in `Allocating` indefinitely | Hardware/BMC problem; readiness retries forever (no timeout) | Diagnose the BMH/BMC; if unrecoverable, the tenant deletes the `BareMetalInstance`. |
| BCM device removed mid-flow | Instance stuck; operator log: `BCM device "<name>" no longer exists in BCM inventory but BareMetalHost CR exists — delete the BareMetalInstance or re-register the device in BCM` | A LiteNode was deleted in BCM after its BMH was created | Re-register the device in BCM, **or** delete the orphaned BMH and the `BareMetalInstance`. |

## Diagnostics

### Prometheus metrics (currently emitted)

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

> **Not yet emitted (planned).** The design also describes a
> `osac_bcm_bmh_readiness_duration_seconds` histogram and Kubernetes events
> (`BCMHostAssigned`, `BCMHostReleased`, `BCMConnectionError`). These are **not
> implemented** in the current release — do not build alerts/runbooks on them yet.
> Use the metrics above, the operator logs, and status below.

### Status and conditions

```bash
oc get baremetalinstance <name> -n <ns> -o jsonpath='{.status.phase}{"\n"}'
oc get baremetalinstance <name> -n <ns> -o yaml | yq '.status.conditions'
```

Remember: the allocation condition only ever reads `Allocating BareMetalInstance`,
`No matching hosts available`, or `BareMetalInstance allocated a host (...)`. A
BCM/BMC problem shows as a *persistent* `Allocating` — the cause is in the logs.

### Key operator log messages

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

### Inspect the BMH (provisioning path)

```bash
# The BMH name equals the BCM hostname
oc get bmh -n <bmh-namespace>
oc get bmh <hostname> -n <bmh-namespace> -o jsonpath='{.status.provisioning.state} {.status.operationalStatus}{"\n"}'
oc get bmh <hostname> -n <bmh-namespace> -o yaml | yq '.status.errorMessage, .spec.bmc.address'
```

### Inspect BCM directly (mTLS)

```bash
curl -k --cert admin.pem --key admin.key -X POST https://<bcm-head>:8081/json \
  -H 'Content-Type: application/json' \
  -d '{"service":"cmdevice","call":"getDevice","args":["<hostname>"]}' | jq '.extra_values'
```
An assigned host has `extra_values.osac_instance_id` set; a free host does not.
(Port `8081` is illustrative — it comes from `bcm.url`.)

## Recovery Procedures

### BCM unreachable / cert expired
No manual recovery is needed for transient outages — all BCM calls and BMH
readiness retry **indefinitely** (no timeout), and instances resume when BCM
returns or the cert is replaced (certwatcher, no restart). Instances remain in
`Allocating`/`Deleting` meanwhile.

### Orphaned on-demand BMH
If a `BareMetalInstance` is gone but its BMH remains (e.g. a device was removed in
BCM mid-flow), confirm it's unused and delete it:

```bash
oc get bmh <hostname> -n <bmh-namespace> -o jsonpath='{.spec.consumerRef}{"\n"}'  # confirm empty
oc delete bmh <hostname> -n <bmh-namespace>
```
`UnassignHost` is idempotent — a null BCM device is treated as already cleaned up.

### Host stuck / never ready
Readiness has **no timeout** — if a host never reaches `available`, the tenant
deletes the `BareMetalInstance`. Deletion clears the BCM reservation and removes
the BMH.

### Temporarily disabling the BCM backend
During a BCM outage you can switch inventory back to Metal3 (see
[Backend Switching](#backend-switching)). **Caveat:** instance *deletions* while
BCM is unreachable require manual BCM cleanup afterward (clear `osac_instance_id`
on the affected devices).

## Backend Switching

Switching the inventory backend type is **not seamless** — each backend uses
different host identifiers and cleanup logic, so a new backend cannot deprovision
instances allocated by the old one.

**To switch away from BCM:**
1. **Drain first.** Delete **all** BCM-backed `BareMetalInstance`s and let each
   finish deleting (this triggers `UnassignHost`: clears BCM `osac_instance_id`,
   deletes the on-demand BMH and its operator-managed BMC Secret). Verify none
   remain.
2. Reconfigure the operator's inventory backend (`bcm.enabled: false` and set the
   replacement backend's values) and upgrade.
3. Restart/roll out the operator.

**Switching *to* BCM:** existing instances from a previous backend are unaffected —
they already have `hostClass` set, so they stay on the management path; only new
instances use BCM.

**If you switch without draining:** orphaned reservations/BMHs are left behind and
must be cleaned up by hand (delete the BMHs; clear `osac_instance_id` in BCM). A
zero-downtime migration tool may be considered in the future.
