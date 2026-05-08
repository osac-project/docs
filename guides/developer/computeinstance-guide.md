# Creating a VM with OSAC

This guide walks through creating a ComputeInstance (virtual machine) using the OSAC CLI,
including the prerequisite networking resources. It assumes you already have a Tenant in
`Ready` state (see [Tenant Setup Guide](tenant-setup.md)).

## Contents

- [Prerequisites](#prerequisites)
- [Step 1: Identify the NetworkClass](#step-1-identify-the-networkclass)
- [Step 2: Create a VirtualNetwork](#step-2-create-a-virtualnetwork)
- [Step 3: Create a Subnet](#step-3-create-a-subnet)
- [Step 4: Create the ComputeInstance](#step-4-create-the-computeinstance)
- [Step 5: Monitor the instance](#step-5-monitor-the-instance)
- [Step 6: Access the console](#step-6-access-the-console)

---

## Prerequisites

- The `osac` CLI is installed and configured (API endpoint, authentication token).
- A Tenant is in `Ready` state (see [Tenant Setup Guide](tenant-setup.md)).
- `oc` CLI configured and logged in to the management cluster (for `oc get vm` monitoring).

---

## Step 1: Identify the NetworkClass

List the available network classes to find the one you will use for your VirtualNetwork:

```bash
osac get networkclass
```

Note the `ID` of the NetworkClass you want to use — you will need it in the next step.

---

## Step 2: Create a VirtualNetwork

Create a VirtualNetwork associated with the NetworkClass. The `--ipv4-cidr` defines the
overall IP address range for the network:

```bash
osac create virtualnetwork \
  --name <virtualnetwork_name> \
  --network-class <networkclass-id> \
  --ipv4-cidr 192.168.0.0/16
```

Note the VirtualNetwork `ID` from the output — you will need it when creating a Subnet.

Wait for the VirtualNetwork to reach `READY` state:

```bash
osac get virtualnetwork
```

---

## Step 3: Create a Subnet

Create a Subnet under the VirtualNetwork. The Subnet CIDR must fall within the
VirtualNetwork's CIDR range (e.g., `192.168.1.0/24` is within `192.168.0.0/16`):

```bash
osac create subnet \
  --name <subnet_name> \
  --virtual-network <virtualnetwork-id> \
  --ipv4-cidr 192.168.1.0/24
```

Note the Subnet `ID` from the output — you will need it in the ComputeInstance spec.

Wait for the Subnet to reach `READY` state:

```bash
osac get subnet
```

---

## Step 4: Create the ComputeInstance

Create a YAML file (e.g., `my-vm.yaml`) with the following content. Replace the `subnet`
value with the Subnet ID from Step 3:

```yaml
'@type': type.googleapis.com/osac.public.v1.ComputeInstance
metadata:
  creators:
    - <tenant_name>
  name: <computeinstance_name>
  tenants:
    - <tenant_name>
spec:
  boot_disk:
    size_gib: 10
  cores: 2
  image:
    source_ref: quay.io/containerdisks/fedora:latest
    source_type: registry
  memory_gib: 2
  subnet: <subnet-id>
  user_data: |
    #cloud-config
    user: <username>
    password: <password>
    chpasswd:
      expire: false
  run_strategy: Always
  template: osac.templates.ocp_virt_vm
  template_parameters:
    exposed_ports:
      '@type': type.googleapis.com/google.protobuf.StringValue
      value: 22/tcp
```

> **Note:** The `user_data` field uses cloud-init format. The example above creates a user
> and disables password expiry, allowing you to log in via the VM console. Adjust the
> username and password as needed.

Create the instance:

```bash
osac create -f my-vm.yaml
```

---

## Step 5: Monitor the instance

Track the ComputeInstance status through the OSAC CLI:

```bash
osac get computeinstance
```

You can also monitor the underlying VM on the Kubernetes cluster:

```bash
oc get vm -A
```

Wait for the ComputeInstance state to reach `RUNNING`.

---

## Step 6: Access the console

Once the ComputeInstance is running, attach to its console:

```bash
osac console computeinstance <computeinstance-id>
```

Log in with the credentials defined in `user_data`

> **Tip:** To exit the console, press `Ctrl+]`.
