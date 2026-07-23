# Metal3 Backend Configuration Guide

This guide walks through configuring the bare-metal-fulfillment-operator to use
Metal3 (BareMetalHost) as the inventory and management backend. It covers
prerequisites, host preparation, and operator configuration.

## Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Preparing BareMetalHost resources](#preparing-baremetalhost-resources)
- [Configuring the operator](#configuring-the-operator)
  - [Inventory Secret](#inventory-secret)
  - [Management Secret](#management-secret)


---

## Overview

The bare-metal-fulfillment-operator supports pluggable backends for host
inventory and management. The Metal3 backend uses BareMetalHost custom resources
(from the [Metal3](https://metal3.io/) BareMetalOperator) as the inventory
source and power management interface.

Use the Metal3 backend when:

- Your bare metal hosts are already managed by the OpenShift BareMetalOperator
- BareMetalHost resources exist in the cluster
- No external provisioning system exists

The operator assigns available BareMetalHost resources to BareMetalInstance
requests, and controls power state through the BareMetalHost API. An AAP template
role handles image provisioning and deprovisioning.

---

## Prerequisites

Before configuring the Metal3 backend, verify the following:

1. **The `baremetal` cluster capability is enabled.** The Cluster Baremetal
   Operator (CBO) and the BareMetalOperator (BMO) are deployed by this
   capability. It is enabled by default, but cluster administrators can disable
   it during installation. If disabled, the cluster cannot provision or manage
   bare-metal nodes. See [Viewing the cluster capabilities](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installation_overview/cluster-capabilities#viewing-cluster-capabilities_cluster-capabilities)
   and [Enabling additional enabled capabilities](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installation_overview/cluster-capabilities#enabling-additional-enabled-capabilities_cluster-capabilities)
   in the OpenShift documentation.

   Verify the capability is enabled:

   ```bash
   oc get clusterversion version \
     -o jsonpath='{.status.capabilities.enabledCapabilities}' | grep -q baremetal \
     && echo "enabled" || echo "disabled"
   ```

2. **A Provisioning CR exists with `watchAllNamespaces` enabled.** On
   installer-provisioned infrastructure (IPI) clusters with `platform: baremetal`,
   the Provisioning CR is created automatically. On other installation types, you
   must create it manually. See [Configuring a provisioning resource](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/scaling-a-user-provisioned-cluster-with-the-bare-metal-operator#configuring-a-provisioning-resource-to-scale-user-provisioned-clusters_scaling-a-user-provisioned-cluster-with-the-bare-metal-operator)
   in the OpenShift documentation.

   By default, the BareMetalOperator only watches BareMetalHost resources in the
   `openshift-machine-api` namespace. OSAC manages hosts in a separate namespace,
   so `watchAllNamespaces` must be enabled.

   If the Provisioning CR does not exist, create it with `watchAllNamespaces`
   already enabled:

   ```yaml
   apiVersion: metal3.io/v1alpha1
   kind: Provisioning
   metadata:
     name: provisioning-configuration
   spec:
     provisioningNetwork: "Disabled"
     watchAllNamespaces: true
   ```

   If the Provisioning CR already exists, patch it:

   ```bash
   oc patch provisioning provisioning-configuration \
     --type merge -p '{"spec":{"watchAllNamespaces": true}}'
   ```

   Verify the setting:

   ```bash
   oc get provisioning provisioning-configuration \
     -o jsonpath='{.spec.watchAllNamespaces}{"\n"}'
   ```

3. **BareMetalHost resources are available** in the target namespace. Hosts must
   be enrolled with BMC credentials and reach the `available` provisioning state
   before OSAC can assign them. See [Creating a BMC secret](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/bare-metal-using-bare-metal-as-a-service#bmo-creating-a-bmc-secret_bare-metal-using-bmaas)
   and [Creating a bare-metal host resource](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/bare-metal-using-bare-metal-as-a-service#bmo-creating-a-bare-metal-host-resource_bare-metal-using-bmaas)
   in the OpenShift documentation.

   ```bash
   oc get baremetalhost -n <namespace>
   ```

4. **The bare-metal-fulfillment-operator is deployed.** This guide covers
   configuring it for the Metal3 backend.

---

## Preparing BareMetalHost resources

OSAC uses labels on BareMetalHost resources to match hosts to BareMetalInstance
requests. Before hosts can be discovered by the operator, an administrator must
apply the required labels.

### Admin-managed labels

#### `osac.openshift.io/host-type`

**Required.** This label identifies the type of hardware (for example, `fc430`,
`gpu-h100`). When a BareMetalInstance request specifies a `hostType`, the
operator filters BareMetalHost resources by this label.

Apply the label to each host:

```bash
oc label baremetalhost <host-name> -n <namespace> \
  osac.openshift.io/host-type=<host-type>
```

For example, to label a host as a GPU node:

```bash
oc label baremetalhost worker-0 -n osac-host-inventory \
  osac.openshift.io/host-type=gpu-h100
```

### System-managed labels and fields

The following are set automatically by the operator during host assignment. Do
not modify them manually.

| Field | Type | Purpose |
|-------|------|---------|
| `osac.openshift.io/managed-by` | Label | Used by profiles to track host provisioning state. Hosts without this label are treated as `managed-by=baremetal`. Do not set manually — it is managed by profile configuration. |
| `osac.openshift.io/pool-id` | Label | Identifies the BareMetalPool that owns the host |
| `spec.consumerRef` | Spec field | Marks the host as claimed by a BareMetalInstance (prevents double-booking by the BareMetalOperator) |

### Host eligibility

A BareMetalHost is eligible for assignment when all of the following conditions
are met:

| Condition | Required value |
|-----------|---------------|
| `status.operationalStatus` | `OK` |
| `status.provisioning.state` | `available` |
| `spec.consumerRef` | Not set (`nil`) |
| `osac.openshift.io/host-type` | Matches the requested `hostType` (if specified) |
| `osac.openshift.io/managed-by` | Defaults to `baremetal` if absent; must match the profile's `hostSelector` (also defaults to `baremetal`) |

To check the state of hosts in the Metal3 namespace:

```bash
oc get baremetalhost -n <namespace> \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.operationalStatus,STATE:.status.provisioning.state,HOST-TYPE:.metadata.labels.osac\\.openshift\\.io/host-type
```

---

## Configuring the operator

The Metal3 backend is configured through two Kubernetes Secrets that the operator
mounts at startup: one for inventory (host discovery and assignment) and one for
management (power control). These Secrets must exist in the operator's namespace
before the operator starts.

The OSAC helm chart references these Secrets by name (configurable via
`bmf.secrets.inventoryConfig` and `bmf.secrets.managementConfig` in
`values.yaml`).

### Inventory Secret

The inventory Secret tells the operator how to discover and assign BareMetalHost
resources:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: osac-inventory-config
  namespace: <operator-namespace>
stringData:
  inventory.yaml: |
    name: bare-metal-inventory
    type: metal3
    options:
      metal3:
        namespace: osac-host-inventory
    hostClass: metal3
    networkClass: metal3
```

```bash
oc apply -f inventory-secret.yaml
```

| Field | Description |
|-------|-------------|
| `name` | A descriptive name for this inventory backend |
| `type` | Must be `metal3` to select the Metal3 inventory backend |
| `options.metal3.namespace` | The Kubernetes namespace where BareMetalHost resources are located |
| `hostClass` | The host class value assigned to hosts after assignment (used by downstream templates) |
| `networkClass` | The network class value assigned to hosts after assignment (used by downstream templates) |

### Management Secret

The management Secret tells the operator how to control power state on
BareMetalHost resources:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: osac-management-config
  namespace: <operator-namespace>
stringData:
  management.yaml: |
    name: bare-metal-management
    type: metal3
    options:
      metal3:
        namespace: osac-host-inventory
```

```bash
oc apply -f management-secret.yaml
```

| Field | Description |
|-------|-------------|
| `name` | A descriptive name for this management backend |
| `type` | Must be `metal3` to select the Metal3 management backend |
| `options.metal3.namespace` | The Kubernetes namespace where BareMetalHost resources are located (must match the inventory namespace) |

The management backend controls power state by patching `spec.online` on the
BareMetalHost. When `spec.online` differs from `status.poweredOn`, a power
transition is in progress and the operator will wait for it to complete before
issuing a new change.

---

## References

- [Bare Metal Fulfillment Enhancement Proposal](https://github.com/osac-project/enhancement-proposals/tree/main/enhancements/bare-metal-fulfillment)
- [AAP Provisioning Architecture](../../architecture/aap-provisioning/)
- [Metal3 Project](https://metal3.io/)
- [BareMetalHost API Reference](https://github.com/metal3-io/baremetal-operator/blob/main/docs/api.md)
- [OpenShift: Creating a BMC secret](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/bare-metal-using-bare-metal-as-a-service#bmo-creating-a-bmc-secret_bare-metal-using-bmaas) — BMC credential setup for BareMetalHost resources
- [OpenShift: Creating a bare-metal host resource](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/bare-metal-using-bare-metal-as-a-service#bmo-creating-a-bare-metal-host-resource_bare-metal-using-bmaas) — enrolling BareMetalHost resources
- [OpenShift: Cluster capabilities](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installation_overview/cluster-capabilities) — viewing and enabling the `baremetal` capability
