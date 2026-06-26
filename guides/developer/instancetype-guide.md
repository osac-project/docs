# Managing Instance Types

This guide walks through managing instance types in OSAC. Instance types are
pre-configured compute bundles (cores, memory) that virtual machines reference
by name. It covers the full lifecycle: creating an instance type, listing and
describing available types, deprecating and obsoleting types that are being
phased out, reactivating types, and deleting types that are no longer needed.

## Contents

- [Overview](#overview)
- [Who does what](#who-does-what)
- [Prerequisites](#prerequisites)
- [Browsing instance types](#browsing-instance-types)
  - [List instance types](#list-instance-types)
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
users choose a named instance type that maps to a specific number of cores
and amount of memory.

| Resource | Purpose | Managed by |
|----------|---------|------------|
| **InstanceType** | Pre-configured compute bundle (cores, memory) that VMs reference by name | Cloud Provider Admin |

Cloud Provider Admins create and manage instance types through the private
admin API. Organization Users browse the available types through the public
API and select one when creating a VM.

---

## Who does what

| Persona | Actions |
|---------|---------|
| **Cloud Provider Admin** | Creates instance types via the private admin API. Lists and describes instance types. Manages the lifecycle: transitions types from ACTIVE to DEPRECATED to OBSOLETE, reactivates types, and deletes types that are no longer referenced. |
| **Organization User** | Lists and describes available instance types (ACTIVE and DEPRECATED) via the public API. Selects an instance type by name when creating a ComputeInstance. |

Organization Users cannot create, modify, or delete instance types.

---

## Prerequisites

- The `osac` CLI is installed and configured (API endpoint, authentication
  token).
- For admin operations: access to the private admin API.
- For user operations: a Tenant in `Ready` state (see
  [Tenant Setup Guide](tenant-setup.md)).

---

## Browsing instance types

The following operations are available to both Cloud Provider Admins and
Organization Users.

### List instance types

List all instance types:

```bash
osac get instancetype
```

The output shows each type in a table with the following columns:

| NAME | CORES | MEMORY | STATE | DESCRIPTION |
|------|-------|--------|-------|-------------|

ACTIVE and DEPRECATED types are shown by default. OBSOLETE types are hidden
from the default listing. To list only OBSOLETE types, filter by state:

```bash
osac get instancetype --filter "state=OBSOLETE"
```

---

### Describe an instance type

View the full details of a specific instance type:

```bash
osac describe instancetype standard-4-16
```

The output shows the instance type's name, cores, memory_gib, state,
description, and deprecation details if the type has been deprecated.

This works for any instance type regardless of state, including OBSOLETE
types that are hidden from the default listing.

---

## Admin operations

### Create an instance type

Create an instance type with the desired compute specification:

```bash
osac create instancetype \
  --name standard-4-16 \
  --cores 4 \
  --memory-gib 16 \
  --description "Balanced compute: 4 cores, 16 GiB RAM"
```

The instance type is immediately available in ACTIVE state. Users can select
it when creating VMs.

---

### Deprecate an instance type

To deprecate an instance type, edit it and change the state:

```bash
osac edit instancetype standard-4-16
```

In the editor, change `spec.state` from `ACTIVE` to `DEPRECATED`. You can
also set the following deprecation fields:

- `spec.deprecation.replacement` -- the name of the replacement instance type
  (e.g., `standard-4-32`).
- `spec.deprecation.obsolescence_timestamp` -- the timestamp when this type
  will become OBSOLETE.

Deprecation and obsolescence timestamps auto-populate when the state
transitions. A DEPRECATED instance type remains available for VM creation,
but users receive a warning indicating the type will be phased out.

---

### Obsolete an instance type

To obsolete an instance type, edit it and change the state:

```bash
osac edit instancetype standard-4-16
```

In the editor, change `spec.state` from `DEPRECATED` to `OBSOLETE`. VMs can
no longer be created with an OBSOLETE instance type. Existing VMs that
reference it are not affected.

---

### Reactivate an instance type

Instance type state transitions are bidirectional. To reactivate a
DEPRECATED or OBSOLETE instance type:

```bash
osac edit instancetype standard-4-16
```

In the editor, change `spec.state` back to `ACTIVE`. The instance type
becomes available for VM creation again. Transitions from OBSOLETE to
DEPRECATED and from DEPRECATED to ACTIVE are both allowed.

---

### Delete an instance type

Delete an instance type that is no longer needed:

```bash
osac delete instancetype standard-4-16
```

Deletion is only allowed when no ComputeInstances reference the instance
type. If any VMs still use it, the request is rejected with a `409 Conflict`
error. See [Troubleshooting](#cannot-delete-an-instance-type) for details.

---

## User operations

### Selecting an instance type for VM creation

When creating a ComputeInstance, specify the instance type using the
`--instance-type` flag:

```bash
osac create computeinstance \
  --name <computeinstance-name> \
  --instance-type standard-4-16 \
  ...
```

For the full VM creation walkthrough, see the
[ComputeInstance Guide](computeinstance-guide.md).

If you select a DEPRECATED instance type, a warning is printed to stderr
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

```bash
osac get computeinstance
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
