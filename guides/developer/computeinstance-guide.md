# Creating a VM with OSAC

This guide walks through creating a ComputeInstance (virtual machine) using the OSAC CLI.
It assumes you already have a Tenant in `Ready` state (see [Tenant Setup Guide](tenant-setup.md))
and networking resources set up (see [Networking Guide](networking-guide.md)).

## Contents

- [Prerequisites](#prerequisites)
- [Step 1: Find a Compute Instance Catalog Item](#step-1-find-a-compute-instance-catalog-item)
- [Step 2: Choose an instance type](#step-2-choose-an-instance-type)
- [Step 3: Create the ComputeInstance](#step-3-create-the-computeinstance)
- [Step 4: Monitor the instance](#step-4-monitor-the-instance)
- [Step 5: Access the console](#step-5-access-the-console)

---

## Prerequisites

- The `osac` CLI is installed and configured (API endpoint, authentication token).
- A Tenant is in `Ready` state (see [Tenant Setup Guide](tenant-setup.md)).
- Networking resources (VirtualNetwork, Subnet, and optionally SecurityGroups) are created
  and in `READY` state (see [Networking Guide](networking-guide.md)).
- At least one compute instance catalog item is published and available
  (see [Compute Instance Catalog Items Guide](computeinstance-catalogitem-guide.md)).
- At least one instance type is in `ACTIVE` state
  (see [Managing Instance Types](instancetype-guide.md)).

---

## Step 1: Find a Compute Instance Catalog Item

Catalog items are curated infrastructure offerings that define defaults and constraints for
compute instances. List the available catalog items:

```bash
osac get computeinstancecatalogitems
```

To inspect a catalog item's details (defaults, editable fields):

```bash
osac get computeinstancecatalogitems <id> -o yaml
```

Note the **NAME** or **ID** of the catalog item you want to use.

> **Note:** Catalog items are created by platform admins. If no catalog items are available,
> contact your platform administrator. For details on catalog item lifecycle management and
> how field definitions control VM creation, see the
> [Compute Instance Catalog Items Guide](computeinstance-catalogitem-guide.md).

---

## Step 2: Choose an instance type

List the available instance types to find the compute configuration for your VM:

```bash
osac get instancetype
```

The output shows each type's name, cores, memory, state, and description.
Choose an ACTIVE instance type that meets your workload requirements.

Note the **NAME** of the instance type you want to use.

> **Note:** This step is optional if the catalog item already defines a default value for
> `instance_type`. You can check the catalog item's `field_definitions` to see
> whether this field has a non-editable default.

For details on instance type lifecycle and management, see
[Managing Instance Types](instancetype-guide.md).

---

## Step 3: Create the ComputeInstance

Create the ComputeInstance using the `osac create computeinstance` command. Use
`--catalog-item` to reference the catalog item and `--network-attachment` for networking.

The catalog item's `field_definitions` determine which fields you can set. Fields marked
as `editable: true` accept your values via CLI flags; fields marked as `editable: false`
are locked to the catalog item's defaults and cannot be overridden. Inspect the catalog
item's field definitions beforehand to see which flags apply (see
[How field definitions shape VM creation](computeinstance-catalogitem-guide.md#how-field-definitions-shape-vm-creation)).

```bash
osac create computeinstance \
  --name <computeinstance_name> \
  --catalog-item <catalog-item-name-or-id> \
  --instance-type <instance-type-name> \
  --image quay.io/containerdisks/fedora:latest \
  --ssh-key "$(cat ~/.ssh/id_ed25519.pub)" \
  --boot-disk-size 10 \
  --network-attachment subnet=<subnet-id> \
  --run-strategy Always \
  --user-data '#cloud-config
user: <username>
password: <password>
chpasswd:
  expire: false'
```

> **Note:** The `--ssh-key` flag installs your SSH public key on the VM, enabling SSH access
> once the instance is running. The `--user-data` field uses cloud-init format. The example
> above creates a user and disables password expiry, allowing you to log in via the VM
> console. Adjust the username and password as needed.
>
> The `--network-attachment` flag uses the format
> `subnet=<subnet-id>[,security-groups=<sg-id1>,<sg-id2>]`. Security groups are optional.
> The flag can be repeated for multiple NICs. See the
> [Networking Guide](networking-guide.md) for details on creating subnets and security
> groups.

### Alternative: Create from a YAML file

You can also create a ComputeInstance from a YAML file with `osac create -f`. Create a file
(e.g., `my-vm.yaml`) with the following content:

```yaml
'@type': type.googleapis.com/osac.public.v1.ComputeInstance
metadata:
  name: <computeinstance_name>
spec:
  catalog_item: <catalog-item-name-or-id>
  instance_type: <instance-type-name>
  image:
    source_ref: quay.io/containerdisks/fedora:latest
    source_type: registry
  ssh_key: <ssh-public-key>
  boot_disk:
    size_gib: 10
  network_attachments:
    - subnet: <subnet-id>
  run_strategy: Always
  user_data: |
    #cloud-config
    user: <username>
    password: <password>
    chpasswd:
      expire: false
```

To attach security groups in the YAML, add them to the network attachment:

```yaml
  network_attachments:
    - subnet: <subnet-id>
      security_groups:
        - <securitygroup-id>
```

Create the instance:

```bash
osac create -f my-vm.yaml
```

---

## Step 4: Monitor the instance

Track the ComputeInstance status through the OSAC CLI:

```bash
osac get computeinstance
```

Wait for the ComputeInstance state to reach `RUNNING`.

---

## Step 5: Access the console

Once the ComputeInstance is running, attach to its serial console:

```bash
osac console serial computeinstance <computeinstance-name-or-id>
```

Log in with the credentials defined in `user_data`.

> **Tip:** To exit the console, press `Ctrl+]`.

To connect via VNC instead:

```bash
osac console vnc computeinstance <computeinstance-name-or-id>
```
