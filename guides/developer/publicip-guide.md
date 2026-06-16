# Assigning a Public IP to a ComputeInstance

This guide walks through allocating a public IP address and attaching it to a
running ComputeInstance so that it becomes reachable from outside the cluster.
It covers the full lifecycle: pool discovery, IP allocation, attachment,
detachment, and release.

For background on why PublicIPs are needed and how they work under the hood,
see the [PublicIP Networking Architecture](../../architecture/publicip-networking.md).

## Contents

- [Overview](#overview)
- [Who does what](#who-does-what)
- [Prerequisites](#prerequisites)
- [Step 1: Find an available PublicIPPool](#step-1-find-an-available-publicippool)
- [Step 2: Allocate a PublicIP](#step-2-allocate-a-publicip)
- [Step 3: Attach the PublicIP to a ComputeInstance](#step-3-attach-the-publicip-to-a-computeinstance)
- [Step 4: Verify connectivity](#step-4-verify-connectivity)
- [Detaching and releasing](#detaching-and-releasing)

---

## Overview

ComputeInstances (VMs) in OSAC use an overlay pod network. By default they can
initiate outbound connections but are not reachable from outside the cluster.
A **PublicIP** gives a VM a routable address on the physical network so that
external clients can connect to it (for example, via SSH or HTTPS).

The PublicIP model uses three resource types:

| Resource | Purpose | Managed by |
|----------|---------|------------|
| **PublicIPPool** | A range of routable IP addresses provisioned by the cloud provider | Cloud Provider Admin |
| **PublicIP** | A single address allocated from a pool, reserved for a tenant | Tenant User |
| **PublicIPAttachment** | Binds a PublicIP to a ComputeInstance, enabling traffic routing | Tenant User |

These resources are independent of each other's lifecycle. Allocating an IP
does not automatically attach it, and detaching does not release it. This
lets tenants reserve addresses and reassign them between instances as needed.

---

## Who does what

| Persona | Actions |
|---------|---------|
| **Cloud Provider Admin** | Creates PublicIPPools via the private admin API. Defines CIDR ranges and IP family (IPv4 or IPv6). |
| **Tenant User** | Discovers available pools (public API). Allocates PublicIPs from a pool. Creates and deletes PublicIPAttachments to route traffic to their ComputeInstances. |

Tenants cannot create or modify pools. They can only allocate IPs from pools
that the provider has made available.

---

## Prerequisites

- The `osac` CLI is installed and configured (API endpoint, authentication token).
- A Tenant is in `Ready` state (see [Tenant Setup Guide](tenant-setup.md)).
- A ComputeInstance is in `RUNNING` state (see [ComputeInstance Guide](computeinstance-guide.md)).
- At least one PublicIPPool has been created by the cloud provider.

> **Tip:** Wherever this guide uses a resource name (e.g., `<pool-name>`),
> you can also use its UUID. The `osac` CLI accepts either form when
> referencing existing resources.

---

## Step 1: Find an available PublicIPPool

List the pools that are visible to your tenant:

```bash
osac get publicippool
```

The output shows each pool's IP family, CIDR ranges, and how many addresses are
still available. Choose a pool that matches the address family you need (IPv4
or IPv6) and has available capacity.

Note the **name** of the pool you want to use.

---

## Step 2: Allocate a PublicIP

Allocate an address from the chosen pool:

```bash
osac create publicip \
  --name <publicip-name> \
  --pool <pool-name>
```

The PublicIP starts in `PENDING` state while the system reserves an address.
Wait for it to reach `ALLOCATED`:

```bash
osac get publicip
```

Once allocated, the output shows the assigned address in the `ADDRESS` column.
The IP is now reserved for your tenant but is not yet routing traffic to any
resource.

> **Note:** The `--pool` field is immutable. A PublicIP cannot be moved to a
> different pool after creation. To use a different pool, delete this PublicIP
> and create a new one.

---

## Step 3: Attach the PublicIP to a ComputeInstance

Create a PublicIPAttachment to bind the allocated IP to a running
ComputeInstance:

```bash
osac create publicipattachment \
  --name <attachment-name> \
  --public-ip <publicip-name> \
  --compute-instance <computeinstance-name>
```

The attachment starts in `PENDING` state while the system configures network
routing. Wait for it to reach `READY`:

```bash
osac get publicipattachment
```

When the attachment is `READY`, external clients can reach the ComputeInstance
at the public IP address. The PublicIP's status also updates to show
`ATTACHED: true`.

> **Constraints:**
> - Each PublicIP can have at most one attachment at a time.
> - Each ComputeInstance can have at most one PublicIPAttachment at a time.
> - Both fields (public IP and compute instance) are immutable. To change
>   the target, delete the attachment and create a new one.

---

## Step 4: Verify connectivity

Test that the ComputeInstance is reachable at the public IP address. For
example, if the VM exposes SSH on port 22:

```bash
ssh <username>@<public-ip-address>
```

Or check with netcat:

```bash
nc -zv <public-ip-address> 22
```

---

## Detaching and releasing

### Detach only (keep the IP reserved)

Delete the PublicIPAttachment to stop routing traffic to the ComputeInstance
while keeping the IP address allocated:

```bash
osac delete publicipattachment <attachment-name>
```

The PublicIP remains in `ALLOCATED` state with `ATTACHED: false`. You can
create a new PublicIPAttachment to attach it to a different ComputeInstance.

### Release the IP (return it to the pool)

First detach the IP if it is currently attached (a PublicIP cannot be deleted
while attached):

```bash
osac delete publicipattachment <attachment-name>
```

Then delete the PublicIP to return the address to the pool:

```bash
osac delete publicip <publicip-name>
```

The address becomes available for other tenants to allocate.
