# Tenant Setup Guide

This guide walks through the steps required to set up a Tenant and create a ComputeInstance
on an OSAC management cluster. It is intended for developers working on or testing the
OSAC operator.

## Contents

- [Concepts](#concepts)
- [First-time setup](#first-time-setup)
- [Returning users: what has changed](#returning-users-what-has-changed)
- [Creating a ComputeInstance](#creating-a-computeinstance)
- [Troubleshooting](#troubleshooting)

---

## Concepts

### Two-cluster model

OSAC distinguishes between two cluster roles:

- **Management cluster**: where the OSAC operator and its CRDs run. Tenant CRs and
  ComputeInstance CRs are created here.
- **Target cluster**: where VMs and their associated resources (namespaces, PVCs,
  StorageClasses) actually run.

Management and target clusters may be the same cluster or different clusters. When the
operator is configured with `OSAC_REMOTE_CLUSTER_KUBECONFIG`, it uses that kubeconfig to
reach the target cluster. When that variable is not set, the operator uses the cluster it is
running on as both management and target.

### Operator namespace

The OSAC operator uses the Kubernetes Downward API to read its own namespace at startup.
Several `OSAC_*_NAMESPACE` variables, including `OSAC_TENANT_NAMESPACE` and
`OSAC_COMPUTE_INSTANCE_NAMESPACE`, are set this way rather than as explicit string values in
the deployment spec. If you inspect the deployment and see `valueFrom.fieldRef.fieldPath:
metadata.namespace` instead of a literal value, that is expected: the pod resolves it to its
own namespace at runtime.

To find your operator namespace:

```bash
oc get pods -A -l control-plane=controller-manager
```

The rest of this guide uses `osac-devel` as a concrete example. Replace it with your actual
operator namespace if it differs.

### What the namespace variables control

- `OSAC_TENANT_NAMESPACE`: the namespace on the management cluster where the operator watches
  for Tenant CRs. Tenant CRs must be created here.
- `OSAC_COMPUTE_INSTANCE_NAMESPACE`: the namespace on the management cluster where the
  operator watches for ComputeInstance CRs. ComputeInstance CRs must be created here.

Both resolve to the operator's own namespace by default. The tenant namespace on the target
cluster (where VMs actually run) is a separate concept: it is determined by the Tenant CR
name, not by these variables.

### Tenant CR and namespace coupling

A Tenant CR on the management cluster is coupled by name to a namespace on the target cluster.
If the Tenant CR is named `my-tenant`, the operator expects a namespace called `my-tenant` to
exist on the target cluster. The names must match exactly.

### Tenant Ready condition

A Tenant CR reaches `Ready` only when two conditions are met on the target cluster:

1. A namespace with the same name as the Tenant CR exists.
2. Exactly one StorageClass labeled `osac.openshift.io/tenant: <tenant-name>` exists.

If either condition is not met, the Tenant stays in `Progressing`.

---

## First-time setup

### Prerequisites

- `oc` CLI configured and logged in to the management cluster.
- If management and target clusters are separate: a second `oc` context (or separate terminal)
  pointed at the target cluster.
- All OSAC CRDs are applied. The networking CRDs (`SecurityGroup`, `Subnet`, `VirtualNetwork`)
  must exist before the operator can start. Verify:

  ```bash
  oc get crd | grep osac.openshift.io
  ```

  Expected CRDs include (at minimum):

  ```
  computeinstances.osac.openshift.io
  tenants.osac.openshift.io
  securitygroups.osac.openshift.io
  subnets.osac.openshift.io
  virtualnetworks.osac.openshift.io
  ```

  If any are missing, apply them from the operator repository before starting the operator:

  ```bash
  oc apply -f config/crd/bases/
  ```

- The OSAC operator is deployed and running:

  ```bash
  oc get pods -n osac-devel -l control-plane=controller-manager
  ```

> **Upgrading from cloudkit-operator?** If your cluster previously ran cloudkit-operator,
> complete the [Migrating from cloudkit-operator](#migrating-from-cloudkit-operator) steps
> before proceeding. The leftover `tenants.cloudkit.openshift.io` CRD will cause `oc get tenant`
> to resolve to the wrong resource throughout this guide.

---

### Step 1: Create the tenant namespace (target cluster)

The namespace name must exactly match the name you will give the Tenant CR in the next step.

```bash
oc create namespace my-tenant
```

Verify:

```bash
oc get namespace my-tenant
```

---

### Step 2: Create the Tenant CR (management cluster)

The Tenant CR goes in the operator namespace (`OSAC_TENANT_NAMESPACE`, which defaults to the
operator's own namespace). A minimal spec is sufficient:

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: Tenant
metadata:
  name: my-tenant
  namespace: osac-devel
spec: {}
```

Apply it:

```bash
oc apply -f tenant.yaml
```

After applying, the Tenant will be in `Progressing` because the StorageClass does not exist
yet. It will not become `Ready` until Step 3 is complete.

---

### Step 3: Create the tenant StorageClass (target cluster)

The operator discovers storage for a tenant by looking for a StorageClass with the label
`osac.openshift.io/tenant: <tenant-name>`. There must be exactly one such StorageClass per
tenant.

The StorageClass provisioner and parameters depend on what storage is available on your
target cluster. The example below uses LVMS (topolvm):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-tenant-default
  labels:
    osac.openshift.io/tenant: my-tenant
provisioner: topolvm.io
parameters:
  csi.storage.k8s.io/fstype: xfs
  topolvm.io/device-class: vg1
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

To find the available LVMS device classes on your cluster:

```bash
oc get lvmcluster -n openshift-storage -o jsonpath='{.items[*].spec.storage.deviceClasses[*].name}'
```

Apply the StorageClass:

```bash
oc apply -f storageclass.yaml
```

---

### Step 4: Verify the Tenant is Ready

Once the namespace and StorageClass are in place, the Tenant should transition to `Ready`:

```bash
oc get tenant my-tenant -n osac-devel \
  -o jsonpath='{.status.phase}{"\n"}'
# Expected: Ready
```

For more detail:

```bash
oc get tenant my-tenant -n osac-devel -o yaml
```

The `status` section will show:

```yaml
status:
  phase: Ready
  namespace: my-tenant
  storageClass: my-tenant-default
```

If it stays `Progressing`, see [Troubleshooting](#troubleshooting).

---

## Returning users: what has changed

If you have worked with OSAC before, you may encounter differences from the setup you
remember. This section describes what has changed and what you need to do to bring an existing
environment up to date.

### Migrating from cloudkit-operator

The OSAC operator was previously named cloudkit-operator. On clusters that ran
cloudkit-operator before the rename, the `tenants.cloudkit.openshift.io` CRD may still be
present. In that case, `oc get tenant` resolves to the wrong CRD.

The clean fix is to delete the old CRD (since cloudkit-operator no longer exists):

```bash
oc delete crd tenants.cloudkit.openshift.io
```

This also deletes any existing `tenants.cloudkit.openshift.io` CRs, so verify there are none
you need to preserve first:

```bash
oc get tenants.cloudkit.openshift.io -A
```

If you cannot delete the CRD, use the fully qualified form for all OSAC tenant commands:

```bash
oc get tenants.osac.openshift.io -n osac-devel
```

---

### Tenant CRs are no longer auto-created

In earlier versions of the operator, Tenant CRs were created automatically. They are now
managed manually by the developer or admin. If you find a Tenant CR that was auto-created
and is in an unexpected state, delete it and follow [Step 2](#step-2-create-the-tenant-cr-management-cluster)
to create it fresh.

```bash
oc delete tenant my-tenant -n osac-devel
```

### StorageClass labeling is now required

The Tenant controller now requires exactly one StorageClass labeled
`osac.openshift.io/tenant: <tenant-name>` to exist before the Tenant can become `Ready`.

If you have an existing Tenant that is stuck in `Progressing` after updating the operator,
check whether a labeled StorageClass exists:

```bash
oc get sc -l osac.openshift.io/tenant=my-tenant
```

If no StorageClass is returned, follow [Step 3](#step-3-create-the-tenant-storageclass-target-cluster)
to create one. If more than one is returned, remove the label from all but one.

### Networking CRDs must be applied when upgrading

The `SecurityGroup`, `Subnet`, and `VirtualNetwork` CRDs were added as part of the
networking work in the OSAC operator. If you are upgrading an existing deployment that
predates this work, these CRDs will not exist on your cluster and the operator will fail to
start because it cannot set up watches for these resource types.

Apply the CRDs before (re)starting the operator:

```bash
oc apply -f config/crd/bases/osac.openshift.io_securitygroups.yaml
oc apply -f config/crd/bases/osac.openshift.io_subnets.yaml
oc apply -f config/crd/bases/osac.openshift.io_virtualnetworks.yaml
```

Also update the operator's ClusterRole to include the new permissions:

```bash
oc apply -f config/rbac/role.yaml
```

After applying CRDs and RBAC, restart the operator:

```bash
oc rollout restart deployment osac-operator-controller-manager -n osac-devel
```

Verify all controllers start:

```bash
oc logs -l control-plane=controller-manager -n osac-devel --tail=50 | grep "Starting Controller"
```

> **Note:** VirtualNetwork and Subnet objects are not yet required for ComputeInstance
> creation to work. The CRDs must exist so the operator can start, but you do not need to
> create any VirtualNetwork or Subnet objects to provision a VM at this time. This guide will
> be updated when network attachment becomes a required step.

---

## Creating a ComputeInstance

Once the Tenant is `Ready`, you can create a ComputeInstance. The CR goes in the operator
namespace (`OSAC_COMPUTE_INSTANCE_NAMESPACE`, which defaults to the operator's own namespace).
This is distinct from the tenant namespace on the target cluster: the ComputeInstance CR is
a management-plane object, and the operator uses it to provision the actual VM in the tenant
namespace.

The `osac.openshift.io/tenant` annotation must match the Tenant CR name exactly.

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: ComputeInstance
metadata:
  name: my-vm
  namespace: osac-devel
  annotations:
    osac.openshift.io/tenant: my-tenant
spec:
  templateID: osac.templates.ocp_virt_vm
  templateParameters: '{}'
  cores: 2
  memoryGiB: 4
  bootDisk:
    sizeGiB: 20
    # storageClass: my-tenant-default  # optional: omit to auto-discover from tenant label
  image:
    sourceType: registry
    sourceRef: quay.io/containerdisks/fedora:latest
  runStrategy: Always
```

> **Note:** The `namespace` field in the metadata is optional if you pass `-n osac-devel` to
> `oc apply`. Including it in the manifest is recommended to avoid accidental creation in the
> wrong namespace.

Apply and watch:

```bash
oc apply -f computeinstance.yaml
oc get computeinstance my-vm -n osac-devel -w
```

The `bootDisk.storageClass` field is optional. When omitted, the operator discovers the
StorageClass automatically from the tenant label. You can also set it explicitly to override
the tenant StorageClass with any other StorageClass available on the target cluster.

---

## Troubleshooting

### Tenant stays in Progressing

Check that both prerequisites are met:

```bash
# Namespace exists on the target cluster
oc get namespace my-tenant

# Exactly one labeled StorageClass exists
oc get sc -l osac.openshift.io/tenant=my-tenant
```

Check the operator logs for the reconcile message:

```bash
oc logs -l control-plane=controller-manager -n osac-devel --tail=50 | grep -i tenant
```

Common log messages:

| Message | Cause |
|---------|-------|
| `namespace not found` | The tenant namespace does not exist on the target cluster |
| `no StorageClass found` | No StorageClass has the tenant label |
| `count: 2` (or higher) | More than one StorageClass has the tenant label |

### Operator controllers do not start

If only some controllers start (check for `Starting Controller` log lines), the operator
may be missing RBAC permissions or CRDs for a resource type. This typically happens after
upgrading to a version that introduced new CRDs.

Diagnose:

```bash
oc logs -l control-plane=controller-manager -n osac-devel --tail=100 | grep -i "failed to watch\|forbidden"
```

Fix by applying the updated RBAC and restarting:

```bash
oc apply -f config/rbac/role.yaml
oc rollout restart deployment osac-operator-controller-manager -n osac-devel
```

### ComputeInstance stays in Starting

Check the operator logs for errors:

```bash
oc logs -l control-plane=controller-manager -n osac-devel --tail=100 | grep -i "computeinstance\|error"
```

Verify the Tenant is Ready:

```bash
oc get tenant my-tenant -n osac-devel \
  -o jsonpath='{.status.phase}{"\n"}'
```
