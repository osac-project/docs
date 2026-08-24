# Managing Instance Types

This guide walks through managing instance types in OSAC. Instance types are
pre-configured compute bundles (cores, memory, and optionally GPU hardware)
that virtual machines reference by name. It covers the full lifecycle:
creating an instance type, listing and describing available types, deprecating
and obsoleting types that are being phased out, reactivating types, and
deleting types that are no longer needed.

Each step shows the `osac` CLI command and, for API-facing operations, the
equivalent gRPC / REST API calls so that the same guide works for both CLI
users and application developers integrating with OSAC's API.

## Contents

- [Overview](#overview)
  - [GPU fields](#gpu-fields)
- [Who does what](#who-does-what)
- [Prerequisites](#prerequisites)
- [Browsing instance types](#browsing-instance-types)
  - [List instance types](#list-instance-types)
  - [Identifying GPU-enabled instance types](#identifying-gpu-enabled-instance-types)
  - [Describe an instance type](#describe-an-instance-type)
- [Admin operations](#admin-operations)
  - [Create an instance type](#create-an-instance-type)
  - [Deprecate an instance type](#deprecate-an-instance-type)
  - [Obsolete an instance type](#obsolete-an-instance-type)
  - [Reactivate an instance type](#reactivate-an-instance-type)
  - [Delete an instance type](#delete-an-instance-type)
- [User operations](#user-operations)
  - [Selecting an instance type for VM creation](#selecting-an-instance-type-for-vm-creation)
- [Troubleshooting](#troubleshooting)

---

## Overview

Instance types define standardized compute configurations that users select
when creating virtual machines. Instead of specifying raw resource values,
users choose a named instance type that maps to a specific number of cores,
amount of memory, and optionally GPU hardware.

| Resource | Purpose | Managed by |
|----------|---------|------------|
| **InstanceType** | Pre-configured compute bundle (cores, memory, optional GPU) that VMs reference by name | Cloud Provider Admin |

Cloud Provider Admins create and manage instance types through the private
admin API. Organization Users browse the available types through the public
API and select one when creating a VM.

### GPU fields

Instance types can optionally include GPU hardware specifications. When
present, the `gpu` field defines the GPU devices that will be attached to
VMs provisioned with this instance type via KubeVirt host device passthrough.

| Field | Type | Description |
|-------|------|-------------|
| `gpu.pci_device_selector` | string | PCI vendor:device ID identifying the GPU hardware (e.g., `10DE:20B0`). Must match the device selector configured on the cluster's GPU nodes. |
| `gpu.resource_name` | string | Kubernetes device plugin resource name (e.g., `nvidia.com/A100`). Used by KubeVirt to request GPU resources from the node. |
| `gpu.count` | int32 | Number of GPU devices of this type (1–16). |

**Validation rules:** When the `gpu` field is present, all three sub-fields
are required. `pci_device_selector` and `resource_name` must be non-empty
strings. `count` must be between 1 and 16.

**Immutability:** The `gpu` field, like `cores` and `memory_gib`, is
immutable after creation. To change the GPU configuration, deprecate the
existing instance type and create a new one.

The Cloud Provider Admin is responsible for entering `pci_device_selector`
and `resource_name` values that match the GPU hardware and device plugins
installed on the cluster nodes. OSAC does not validate these values against
the cluster's actual hardware inventory.

---

## Who does what

| Persona | Actions |
|---------|---------|
| **Cloud Provider Admin** | Creates instance types via the private admin API. Lists and describes instance types. Manages the lifecycle: transitions types from ACTIVE to DEPRECATED to OBSOLETE, reactivates types, and deletes types that are no longer referenced. |
| **Organization User** | Lists and describes available instance types (ACTIVE and DEPRECATED) via the public API. Selects an instance type by name when creating a ComputeInstance. |

Organization Users cannot create, modify, or delete instance types.

---

## Prerequisites

**CLI users:**

- The `osac` CLI is installed and configured (API endpoint, authentication
  token).

**API users:**

- `grpcurl` (for gRPC) or `curl` (for REST) is installed.
- Set the following environment variables for the examples in this guide:

  ```bash
  export OSAC_API="<api-endpoint>"    # host:port (e.g., fulfillment-api.osac.svc:443)
  export TOKEN="<authentication-token>"

  # Skip TLS verification in development environments:
  # export GRPCURL_FLAGS="-insecure"
  # export CURL_FLAGS="-k"
  ```

**Both:**

- For admin operations: access to the private admin API.
- For user operations: a Tenant in `Ready` state (see
  [Tenant Setup Guide](tenant-setup.md)).

---

## Browsing instance types

The following operations are available to both Cloud Provider Admins and
Organization Users.

### List instance types

List all instance types:

**CLI:**

```bash
osac get instancetype
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  $OSAC_API osac.public.v1.InstanceTypes/List
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  "https://$OSAC_API/api/fulfillment/v1/instance_types"
```

The output shows each type in a table with the following columns:

| NAME | CORES | MEMORY | GPUS | GPU NAME | STATE | DESCRIPTION |
|------|-------|--------|------|----------|-------|-------------|

GPU-enabled instance types show the GPU count and resource name. Non-GPU
types show `0` and `-` in the GPU columns.

ACTIVE and DEPRECATED types are shown by default. OBSOLETE types are hidden
from the default listing. To list only OBSOLETE types, filter by state:

**CLI:**

```bash
osac get instancetype --filter "state=OBSOLETE"
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  -d '{"filter": "state=OBSOLETE"}' \
  $OSAC_API osac.public.v1.InstanceTypes/List
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  "https://$OSAC_API/api/fulfillment/v1/instance_types?filter=state%3DOBSOLETE"
```

---

### Identifying GPU-enabled instance types

The CLI table output includes GPUS and GPU NAME columns. GPU-enabled
instance types show the GPU count and resource name, while non-GPU types
show `0` and `-`.

To list only GPU-enabled instance types, use the `has(this.spec.gpu)` CEL
filter:

**CLI:**

```bash
osac get instancetype --filter "has(this.spec.gpu)"
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  -d '{"filter": "has(this.spec.gpu)"}' \
  $OSAC_API osac.public.v1.InstanceTypes/List
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  "https://$OSAC_API/api/fulfillment/v1/instance_types?filter=has(this.spec.gpu)"
```

To filter by a specific GPU resource name:

```bash
osac get instancetype --filter "this.spec.gpu.resource_name == 'nvidia.com/A100'"
```

---

### Describe an instance type

View the full details of a specific instance type:

**CLI:**

```bash
osac describe instancetype standard-4-16
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  -d '{"id": "standard-4-16"}' \
  $OSAC_API osac.public.v1.InstanceTypes/Get
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  "https://$OSAC_API/api/fulfillment/v1/instance_types/standard-4-16"
```

The output shows the instance type's name, cores, memory_gib, state,
description, and deprecation details if the type has been deprecated. For
GPU-enabled instance types, the output also includes the `gpu` field with
`pci_device_selector`, `resource_name`, and `count`.

This works for any instance type regardless of state, including OBSOLETE
types that are hidden from the default listing.

---

## Admin operations

### Create an instance type

Create an instance type with the desired compute specification:

**CLI:**

```bash
osac create instancetype \
  --name standard-4-16 \
  --cores 4 \
  --memory-gib 16 \
  --description "Balanced compute: 4 cores, 16 GiB RAM"
```

> **Note:** Instance type management uses the **private** admin API
> (`osac.private.v1`). The examples below use the private API service name.

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" -d '{
  "object": {
    "id": "standard-4-16",
    "metadata": {
      "name": "standard-4-16"
    },
    "spec": {
      "cores": 4,
      "memory_gib": 16,
      "description": "Balanced compute: 4 cores, 16 GiB RAM"
    }
  }
}' $OSAC_API osac.private.v1.InstanceTypes/Create
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{
  "id": "standard-4-16",
  "metadata": {
    "name": "standard-4-16"
  },
  "spec": {
    "cores": 4,
    "memory_gib": 16,
    "description": "Balanced compute: 4 cores, 16 GiB RAM"
  }
}' "https://$OSAC_API/api/fulfillment/v1/instance_types"
```

The instance type is immediately available in ACTIVE state. Users can select
it when creating VMs.

#### GPU-enabled instance types

To create an instance type with GPU hardware, include the `gpu` field with
`pci_device_selector`, `resource_name`, and `count`. See
[GPU fields](#gpu-fields) for validation rules and immutability.

**CLI:**

```bash
osac create instancetype \
  --name gpu-a100-8core \
  --cores 8 \
  --memory-gib 64 \
  --gpu-pci-device-selector "10DE:20B0" \
  --gpu-resource-name "nvidia.com/A100" \
  --gpu-count 1 \
  --description "8 vCPU, 64 GiB, 1x A100 GPU"
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" -d '{
  "object": {
    "id": "gpu-a100-8core",
    "metadata": {
      "name": "gpu-a100-8core"
    },
    "spec": {
      "cores": 8,
      "memory_gib": 64,
      "description": "8 vCPU, 64 GiB, 1x A100 GPU",
      "gpu": {
        "pci_device_selector": "10DE:20B0",
        "resource_name": "nvidia.com/A100",
        "count": 1
      }
    }
  }
}' $OSAC_API osac.private.v1.InstanceTypes/Create
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{
  "id": "gpu-a100-8core",
  "metadata": {
    "name": "gpu-a100-8core"
  },
  "spec": {
    "cores": 8,
    "memory_gib": 64,
    "description": "8 vCPU, 64 GiB, 1x A100 GPU",
    "gpu": {
      "pci_device_selector": "10DE:20B0",
      "resource_name": "nvidia.com/A100",
      "count": 1
    }
  }
}' "https://$OSAC_API/api/fulfillment/v1/instance_types"
```

> **Important:** The `pci_device_selector` (e.g., `10DE:20B0`) and
> `resource_name` (e.g., `nvidia.com/A100`) must match the GPU hardware and
> device plugins configured on the cluster's worker nodes. Incorrect values
> cause provisioning failures at the KubeVirt scheduling layer. See
> [Troubleshooting](#computeinstance-stuck-in-provisioning-state-gpu) for
> diagnosis steps.

---

### Deprecate an instance type

To deprecate an instance type, edit it and change the state:

**CLI:**

```bash
osac edit instancetype standard-4-16
```

In the editor, change `spec.state` from `ACTIVE` to `DEPRECATED`. You can
also set the following deprecation fields:

- `spec.deprecation.replacement` -- the name of the replacement instance type
  (e.g., `standard-4-32`).
- `spec.deprecation.obsolescence_timestamp` -- the timestamp when this type
  will become OBSOLETE.

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" -d '{
  "object": {
    "id": "standard-4-16",
    "spec": {
      "state": "INSTANCE_TYPE_STATE_DEPRECATED",
      "deprecation": {
        "replacement": "standard-4-32"
      }
    }
  },
  "update_mask": {
    "paths": ["spec.state", "spec.deprecation"]
  }
}' $OSAC_API osac.private.v1.InstanceTypes/Update
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{
  "spec": {
    "state": "INSTANCE_TYPE_STATE_DEPRECATED",
    "deprecation": {
      "replacement": "standard-4-32"
    }
  }
}' "https://$OSAC_API/api/fulfillment/v1/instance_types/standard-4-16?update_mask=spec.state,spec.deprecation"
```

Deprecation and obsolescence timestamps auto-populate when the state
transitions. A DEPRECATED instance type remains available for VM creation,
but users receive a warning indicating the type will be phased out.

---

### Obsolete an instance type

To obsolete an instance type, edit it and change the state:

**CLI:**

```bash
osac edit instancetype standard-4-16
```

In the editor, change `spec.state` from `DEPRECATED` to `OBSOLETE`. VMs can
no longer be created with an OBSOLETE instance type. Existing VMs that
reference it are not affected.

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" -d '{
  "object": {
    "id": "standard-4-16",
    "spec": {
      "state": "INSTANCE_TYPE_STATE_OBSOLETE"
    }
  },
  "update_mask": {
    "paths": ["spec.state"]
  }
}' $OSAC_API osac.private.v1.InstanceTypes/Update
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{
  "spec": {
    "state": "INSTANCE_TYPE_STATE_OBSOLETE"
  }
}' "https://$OSAC_API/api/fulfillment/v1/instance_types/standard-4-16?update_mask=spec.state"
```

---

### Reactivate an instance type

Instance type state transitions are bidirectional. To reactivate a
DEPRECATED or OBSOLETE instance type:

**CLI:**

```bash
osac edit instancetype standard-4-16
```

In the editor, change `spec.state` back to `ACTIVE`. The instance type
becomes available for VM creation again. Transitions from OBSOLETE to
DEPRECATED and from DEPRECATED to ACTIVE are both allowed.

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" -d '{
  "object": {
    "id": "standard-4-16",
    "spec": {
      "state": "INSTANCE_TYPE_STATE_ACTIVE"
    }
  },
  "update_mask": {
    "paths": ["spec.state"]
  }
}' $OSAC_API osac.private.v1.InstanceTypes/Update
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{
  "spec": {
    "state": "INSTANCE_TYPE_STATE_ACTIVE"
  }
}' "https://$OSAC_API/api/fulfillment/v1/instance_types/standard-4-16?update_mask=spec.state"
```

---

### Delete an instance type

Delete an instance type that is no longer needed:

**CLI:**

```bash
osac delete instancetype standard-4-16
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  -d '{"id": "standard-4-16"}' \
  $OSAC_API osac.private.v1.InstanceTypes/Delete
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -X DELETE -H "Authorization: Bearer $TOKEN" \
  "https://$OSAC_API/api/fulfillment/v1/instance_types/standard-4-16"
```

Deletion is only allowed when no ComputeInstances reference the instance
type. If any VMs still use it, the request is rejected with a `409 Conflict`
error. See [Troubleshooting](#cannot-delete-an-instance-type) for details.

---

## User operations

### Selecting an instance type for VM creation

When creating a ComputeInstance, specify the instance type using the
`--instance-type` flag (CLI) or the `spec.instance_type` field (API):

**CLI:**

```bash
osac create computeinstance \
  --name <computeinstance-name> \
  --instance-type standard-4-16 \
  ...
```

**gRPC / REST:** See the `spec.instance_type` field in the
[ComputeInstance Guide](computeinstance-guide.md#step-3-create-the-computeinstance).

If you select a DEPRECATED instance type, a warning is returned
indicating the type will be phased out and suggesting the replacement (if one
is configured). The VM is still created successfully.

---

## Troubleshooting

### Cannot delete an instance type

```
Error: instance type is in use (409 Conflict)
```

The instance type is referenced by one or more ComputeInstances. Delete or
migrate all VMs using this instance type before deleting it. To find which
VMs are running:

**CLI:**

```bash
osac get computeinstance
```

**gRPC:**

```bash
grpcurl $GRPCURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  $OSAC_API osac.public.v1.ComputeInstances/List
```

**REST:**

```bash
curl -fsS $CURL_FLAGS -H "Authorization: Bearer $TOKEN" \
  "https://$OSAC_API/api/fulfillment/v1/compute_instances"
```

Review the output and delete or recreate the VMs that reference the instance
type you want to remove.

---

### Cannot create a VM with an instance type

**409 Conflict -- instance type is OBSOLETE:**

The instance type has been obsoleted and can no longer be used for new VMs.
List the available instance types to find an ACTIVE replacement:

```bash
osac get instancetype
```

If the instance type has a configured replacement, the describe output shows
the suggested alternative:

```bash
osac describe instancetype <instance-type-name>
```

**404 Not Found -- instance type does not exist:**

The specified instance type name does not match any existing type. Verify the
name by listing available instance types:

```bash
osac get instancetype
```

---

### Deprecation warning when creating a VM

```
Warning: instance type "standard-4-16" is deprecated
```

This is informational -- the VM creation succeeds. The warning indicates that
the instance type will become obsolete in the future. Check the deprecation
details for the suggested replacement:

```bash
osac describe instancetype standard-4-16
```

If a replacement is configured, the output shows the replacement name and the
planned obsolescence date. Consider migrating future VMs to the replacement
instance type.
