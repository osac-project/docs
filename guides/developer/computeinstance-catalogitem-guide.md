# Managing Compute Instance Catalog Items

This guide covers catalog items for compute instances: what they are, how
admins create and manage them, and how users create VMs from them. It explains
the relationship between a catalog item's `field_definitions` and the
arguments accepted when creating a ComputeInstance.

## Contents

- [Overview](#overview)
- [Who does what](#who-does-what)
- [Prerequisites](#prerequisites)
- [Browsing catalog items](#browsing-catalog-items)
  - [List catalog items](#list-catalog-items)
  - [Inspect a catalog item](#inspect-a-catalog-item)
- [Admin operations](#admin-operations)
  - [Create a catalog item](#create-a-catalog-item)
  - [Publish a catalog item](#publish-a-catalog-item)
  - [Update a catalog item](#update-a-catalog-item)
  - [Unpublish a catalog item](#unpublish-a-catalog-item)
  - [Delete a catalog item](#delete-a-catalog-item)
- [How field definitions shape VM creation](#how-field-definitions-shape-vm-creation)
  - [Available paths](#available-paths)
  - [Editable vs non-editable fields](#editable-vs-non-editable-fields)
  - [Defaults and validation](#defaults-and-validation)
  - [Instance type](#instance-type)
  - [Server-side processing](#server-side-processing)
- [User operations](#user-operations)
  - [Create a VM from a catalog item](#create-a-vm-from-a-catalog-item)
- [Troubleshooting](#troubleshooting)

---

## Overview

OSAC uses a three-level hierarchy to provision compute instances:

```text
Template (admin-only)
    ↓  referenced by
Catalog Item (admin-managed, published to users)
    ↓  used by
ComputeInstance (user-created VM)
```

**Templates** are infrastructure blueprints managed through the admin API.
They define the underlying provisioning logic. Templates are not visible to
end users.

**Catalog items** reference a template and add a curation layer. Using
`field_definitions`, the admin controls which ComputeInstance spec fields
users can set, locks fields to fixed values, provides defaults, and
optionally validates user input with JSON Schema. Once published
(`published: true`), catalog items become visible through the public API.

**ComputeInstances** are the VMs that users create. When a user creates a VM
with `--catalog-item`, the server resolves the underlying template from the
catalog item and enforces the catalog item's `field_definitions` against the
user's request.

| Resource | Purpose | Managed by |
|----------|---------|------------|
| **ComputeInstanceTemplate** | Infrastructure blueprint for VM provisioning | Cloud Provider Admin |
| **ComputeInstanceCatalogItem** | Curated offering with field restrictions and defaults | Cloud Provider Admin |
| **ComputeInstance** | The running VM | Organization User |

---

## Who does what

| Persona | Actions |
|---------|---------|
| **Cloud Provider Admin** | Creates templates and catalog items. Publishes, updates, unpublishes, and deletes catalog items. Defines which fields users can control via `field_definitions`. |
| **Organization User** | Lists published catalog items. Creates VMs using `--catalog-item`. Provides values for editable fields and accepts defaults for the rest. |

Organization Users cannot create, modify, or delete catalog items.

---

## Prerequisites

- The `osac` CLI is installed and configured (API endpoint, authentication
  token).
- For admin operations: access to the private admin API.
- For user operations: a Tenant in `Ready` state (see
  [Tenant Setup Guide](tenant-setup.md)).

---

## Browsing catalog items

The following operations are available to both Cloud Provider Admins and
Organization Users.

### List catalog items

List all published catalog items:

```bash
osac get computeinstancecatalogitems
```

The output shows each catalog item in a table:

| ID | NAME | TITLE | PUBLISHED |
|----|------|-------|-----------|

> **Note:** The public API only returns published catalog items. Admins using
> the private API see all catalog items regardless of publish state.

### Inspect a catalog item

View the full details of a catalog item, including its `field_definitions`:

```bash
osac get computeinstancecatalogitems <name-or-id> -o yaml
```

The output shows the catalog item's metadata, title, description, template
reference, publish state, and the complete list of field definitions with
their paths, editability, defaults, and validation schemas.

---

## Admin operations

### Create a catalog item

Catalog items are created using `osac create -f` with a YAML file. The
`field_definitions` structure is too complex for CLI flags, so a file is
required.

First, find the available compute instance templates:

```bash
osac get computeinstancetemplates
```

Create a YAML file (e.g., `standard-vm-catalog-item.yaml`):

```yaml
'@type': type.googleapis.com/osac.public.v1.ComputeInstanceCatalogItem
metadata:
  name: standard-linux-vm
title: Standard Linux VM
description: |
  General-purpose Linux virtual machine with sensible defaults.
  Users can choose their SSH key, instance type, and boot disk size.
template: "osac.templates.ocp_virt_vm"
published: false
field_definitions:
  - path: ssh_key
    display_name: SSH Public Key
    editable: true
  - path: instance_type
    display_name: Instance Type
    editable: true
    default: "standard-4-16"
  - path: image.source_type
    display_name: Image Source Type
    editable: false
    default: "registry"
  - path: image.source_ref
    display_name: Image
    editable: true
    default: "quay.io/containerdisks/fedora:latest"
  - path: boot_disk.size_gib
    display_name: Boot Disk Size (GiB)
    editable: true
    default: 20
    validation_schema: '{"type":"number","minimum":10,"maximum":500}'
  - path: run_strategy
    display_name: Run Strategy
    editable: false
    default: "Always"
  - path: user_data
    display_name: Cloud-init User Data
    editable: true
```

Create the catalog item:

```bash
osac create -f standard-vm-catalog-item.yaml
```

The catalog item is created with `published: false`. It is not visible to
users until you publish it.

> **Note:** The YAML file can contain multiple documents separated by `---`,
> allowing you to create several catalog items in one command.

---

### Publish a catalog item

Publishing makes the catalog item visible to users in the public API. Edit
the catalog item and set `published: true`:

```bash
osac edit computeinstancecatalogitems standard-linux-vm
```

In the editor, change `published` from `false` to `true`, then save and
exit.

---

### Update a catalog item

To modify a catalog item's title, description, field definitions, or
template:

```bash
osac edit computeinstancecatalogitems standard-linux-vm
```

This opens the catalog item in `$EDITOR`. Make your changes, save, and exit.

Changes to `field_definitions` take effect for new VM requests immediately.
Existing VMs are not affected — the catalog item reference on a
ComputeInstance is immutable after creation.

---

### Unpublish a catalog item

To hide a catalog item from users without deleting it, edit it and set
`published: false`:

```bash
osac edit computeinstancecatalogitems standard-linux-vm
```

Unpublished catalog items are no longer returned by the public API and cannot
be used for new VM creation. Existing VMs that reference the catalog item
continue to work normally.

> **Note:** Existing VMs that reference an unpublished catalog item continue
> to run normally. Users can view the catalog item details on their existing
> ComputeInstance resources, but the catalog item itself is no longer returned
> by the public List or Get endpoints.

---

### Delete a catalog item

```bash
osac delete computeinstancecatalogitems standard-linux-vm
```

---

## How field definitions shape VM creation

Each entry in `field_definitions` controls a single field on the
ComputeInstance spec. Together, they define the contract between the catalog
item and the user: which fields the user can set, which are locked, what
defaults apply, and what validation constraints exist.

### Available paths

The following paths can be used in `field_definitions`. These correspond
to fields in the ComputeInstance spec. Any path not in this list is silently
ignored.

| Path | CLI flag | Description |
|------|----------|-------------|
| `ssh_key` | `--ssh-key` | SSH public key installed on the VM |
| `instance_type` | `--instance-type` | Named instance type (e.g., `standard-4-16`) |
| `run_strategy` | `--run-strategy` | VM run strategy (`Always`, `Halted`, etc.) |
| `user_data` | `--user-data` | Cloud-init or ignition user data |
| `image.source_type` | (part of `--image`) | Image source type (e.g., `registry`) |
| `image.source_ref` | `--image` | Image reference (e.g., OCI image URL) |
| `boot_disk.size_gib` | `--boot-disk-size` | Boot disk size in GiB |
| `additional_disks` | — | Additional disk configurations |
| `network_attachments` | `--network-attachment` | Network attachments (subnet + security groups per NIC) |

These paths use dot notation for nested fields. For example,
`boot_disk.size_gib` refers to the `size_gib` field inside the `boot_disk`
object in the ComputeInstance spec.

---

### Editable vs non-editable fields

Each field definition has an `editable` flag that controls whether the user
can provide their own value:

| `editable` | User provides value | Result |
|------------|---------------------|--------|
| `false` | Yes (via CLI flag) | User's value is **silently overridden** by the catalog item default |
| `false` | No | Catalog item default is applied |
| `true` | Yes | User's value is accepted (validated against `validation_schema` if present) |
| `true` | No | Catalog item default is applied (error if no default is defined) |

Non-editable fields enforce consistency. For example, locking `run_strategy`
to `Always` ensures all VMs from this catalog item are always running,
regardless of what the user passes on the CLI.

Editable fields give users flexibility. For example, making `ssh_key`
editable lets each user provide their own public key.

---

### Defaults and validation

**Defaults**: Every non-editable field must have a `default`. Editable fields
should also have a `default` unless the field is required and must always be
provided by the user (like `ssh_key`). If an editable field has no default
and the user does not provide a value, the request fails.

**Validation**: Editable fields can include a `validation_schema` — a JSON
Schema (draft 2020-12) string. The server validates the user's value against
this schema. Non-editable fields do not need validation since their value
always comes from the default.

Example — constrain boot disk size to between 10 and 500 GiB:

```yaml
- path: boot_disk.size_gib
  display_name: Boot Disk Size (GiB)
  editable: true
  default: 20
  validation_schema: '{"type":"number","minimum":10,"maximum":500}'
```

Example — restrict image references to a specific registry:

```yaml
- path: image.source_ref
  display_name: Image
  editable: true
  default: "quay.io/containerdisks/fedora:latest"
  validation_schema: '{"type":"string","pattern":"^quay\\.io/containerdisks/"}'
```

---

### Instance type

Catalog items use the `instance_type` path to control which compute
configuration users select. Instance types are named bundles of cores and
memory managed by the admin (see
[Managing Instance Types](instancetype-guide.md)).

If the catalog item defines `instance_type` with a default value, the server
validates that the referenced instance type exists and is not OBSOLETE. If
it is DEPRECATED, the server returns a warning.

---

### Server-side processing

When a user creates a ComputeInstance with `--catalog-item`, the server
processes the request in this order:

1. **Lookup**: Finds the catalog item by name or ID.
2. **Access check**: Verifies the catalog item is published and not deleted.
3. **Template resolution**: Sets the ComputeInstance's `template` to the
   template referenced by the catalog item.
4. **Apply field definitions**: For each field definition:
   - **Non-editable**: overrides any user-provided value with the default.
   - **Editable with user value**: validates against `validation_schema` if
     present.
   - **Editable without user value**: applies the default.
5. **Validate**: Validates the resulting spec (instance type state, network
   attachments, etc.) and creates the resource.

Fields not covered by any field definition pass through unchanged — the user
can set them freely via CLI flags.

---

## User operations

### Create a VM from a catalog item

List the available catalog items to find one that fits your workload:

```bash
osac get computeinstancecatalogitems
```

Inspect the catalog item to see which fields you can customize:

```bash
osac get computeinstancecatalogitems <name-or-id> -o yaml
```

In the output, review the `field_definitions`:
- Fields with `editable: true` can be set via CLI flags.
- Fields with `editable: false` are locked to the catalog item's default —
  you cannot override them.
- Fields with a `validation_schema` must match the specified constraints.

Create a VM, providing values for the editable fields:

```bash
osac create computeinstance \
  --name my-vm \
  --catalog-item standard-linux-vm \
  --instance-type standard-4-16 \
  --ssh-key "$(cat ~/.ssh/id_ed25519.pub)" \
  --boot-disk-size 40 \
  --network-attachment subnet=<subnet-id> \
  --user-data '#cloud-config
user: myuser
password: <password>
chpasswd:
  expire: false'
```

The server applies the catalog item's field definitions to your request.
Non-editable fields use the catalog item's defaults regardless of what you
pass. Editable fields accept your values, falling back to catalog item
defaults for anything you omit.

For the full VM creation walkthrough, see the
[ComputeInstance Guide](computeinstance-guide.md).

---

## Troubleshooting

### Catalog item not found

```text
Error: catalog item 'my-catalog-item' is not published (404 Not Found)
```

The catalog item either does not exist or is not published. Ask your platform
administrator to verify the catalog item exists and has `published: true`.

### CLI flag ignored for a non-editable field

If you pass a CLI flag for a field that the catalog item marks as
non-editable, the server silently overrides your value with the catalog
item's default. There is no error or warning — inspect the catalog item's
field definitions to see which fields are locked:

```bash
osac get computeinstancecatalogitems <name-or-id> -o yaml
```

### Validation failed for a field

```text
Error: validation failed for field 'boot_disk.size_gib': ...
```

The value you provided does not match the catalog item's `validation_schema`
for that field. Inspect the catalog item to see the constraints:

```bash
osac get computeinstancecatalogitems <name-or-id> -o yaml
```

Look at the `validation_schema` for the field mentioned in the error to
understand the allowed values.

### Required field missing

```text
Error: field 'ssh_key' is required but no value was provided and no default is defined
```

The catalog item has an editable field with no default, and you did not
provide a value. Add the corresponding CLI flag to your request (in this
example, `--ssh-key`).

### Instance type is obsolete

```text
Error: instance type 'standard-4-16' in field_definitions is obsolete and cannot be used
```

The catalog item's default instance type has been obsoleted. Update the
catalog item to reference an ACTIVE instance type:

```bash
osac edit computeinstancecatalogitems <name-or-id>
```

List the available instance types to find a replacement:

```bash
osac get instancetype
```
