# Setting Up Networking with OSAC

This guide walks through creating the networking resources required before provisioning
compute instances: VirtualNetworks and Subnets. It also covers SecurityGroups, which are
optional and only needed when you want to apply firewall rules. It assumes you already
have a Tenant in `Ready` state (see [Tenant Setup Guide](tenant-setup.md)).

## Contents

- [Prerequisites](#prerequisites)
- [Overview](#overview)
- [Step 1: Identify the NetworkClass](#step-1-identify-the-networkclass)
- [Step 2: Create a VirtualNetwork](#step-2-create-a-virtualnetwork)
- [Step 3: Create a Subnet](#step-3-create-a-subnet)
- [Step 4: Create a SecurityGroup (optional)](#step-4-create-a-securitygroup-optional)
- [Next steps](#next-steps)

---

## Prerequisites

- The `osac` CLI is installed and configured (API endpoint, authentication token).
- A Tenant is in `Ready` state (see [Tenant Setup Guide](tenant-setup.md)).

---

## Overview

OSAC networking follows a hierarchical model:

```text
NetworkClass (platform-defined, read-only)
└── VirtualNetwork (tenant L2 network with CIDR)
      ├── Subnet (CIDR range within VirtualNetwork)
      └── SecurityGroup (optional — firewall rules scoped to VirtualNetwork)
```

- A **NetworkClass** is a platform-level resource that defines the type of network
  infrastructure available. It is read-only for tenants.
- A **VirtualNetwork** is a tenant-scoped L2 network associated with a NetworkClass.
  It defines an overall IPv4 CIDR range.
- A **Subnet** is a CIDR range within a VirtualNetwork. Compute instances attach to
  subnets via network attachments.
- A **SecurityGroup** defines ingress and egress firewall rules scoped to a
  VirtualNetwork. Security groups follow a **default-deny** policy — traffic is blocked
  unless a rule explicitly allows it. Rules are stateful, so return traffic for
  established connections is automatically allowed.

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

Note the VirtualNetwork `ID` from the output — you will need it when creating Subnets
and SecurityGroups.

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

Note the Subnet `ID` from the output — you will need it when creating compute instances.

Wait for the Subnet to reach `READY` state:

```bash
osac get subnet
```

---

## Step 4: Create a SecurityGroup (optional)

SecurityGroups are optional. Create one only when you need firewall rules to control
traffic to and from compute instances. They are scoped to a VirtualNetwork and follow a
**default-deny** policy: all traffic is blocked unless explicitly allowed by a rule.

### Create a security group with rules

Use `--ingress` and `--egress` flags to define rules. Each rule is a comma-separated
list of `key=value` pairs:

```bash
osac create securitygroup \
  --name <securitygroup_name> \
  --virtual-network <virtualnetwork-id> \
  --ingress protocol=tcp,port-from=22,port-to=22,ipv4-cidr=0.0.0.0/0 \
  --ingress protocol=tcp,port-from=80,port-to=80,ipv4-cidr=0.0.0.0/0 \
  --ingress protocol=tcp,port-from=443,port-to=443,ipv4-cidr=0.0.0.0/0 \
  --egress protocol=all,ipv4-cidr=0.0.0.0/0
```

The example above allows inbound SSH (port 22), HTTP (port 80), and HTTPS (port 443)
from any IPv4 address, and permits all outbound traffic.

Note the SecurityGroup `ID` from the output — you will need it when creating compute
instances.

Wait for the SecurityGroup to reach `READY` state:

```bash
osac get securitygroup
```

### Rule format

Each `--ingress` or `--egress` flag takes a rule in `key=value,...` format with the
following keys:

| Key | Required | Description |
|-----|----------|-------------|
| `protocol` | Yes | One of: `tcp`, `udp`, `icmp`, `all` |
| `port-from` | For `tcp`/`udp` | Start of port range (1–65535) |
| `port-to` | For `tcp`/`udp` | End of port range (1–65535), must be >= `port-from` |
| `ipv4-cidr` | No | IPv4 CIDR block (e.g., `0.0.0.0/0` for all, `10.0.0.0/8` for private) |
| `ipv6-cidr` | No | IPv6 CIDR block (e.g., `::/0` for all) |

- For `icmp` and `all` protocols, port fields are ignored.
- Both `ipv4-cidr` and `ipv6-cidr` can be set on a single rule to create a dual-stack
  rule.
- The `--ingress` and `--egress` flags can each be repeated to add multiple rules.

### Examples

Allow only SSH from a specific network:

```bash
osac create securitygroup \
  --name ssh-only \
  --virtual-network <virtualnetwork-id> \
  --ingress protocol=tcp,port-from=22,port-to=22,ipv4-cidr=10.0.0.0/8
```

Allow all traffic (open security group):

```bash
osac create securitygroup \
  --name allow-all \
  --virtual-network <virtualnetwork-id> \
  --ingress protocol=all,ipv4-cidr=0.0.0.0/0 \
  --egress protocol=all,ipv4-cidr=0.0.0.0/0
```

Allow a range of UDP ports:

```bash
osac create securitygroup \
  --name udp-range \
  --virtual-network <virtualnetwork-id> \
  --ingress protocol=udp,port-from=5000,port-to=5100,ipv4-cidr=0.0.0.0/0
```

### Inspect a security group

```bash
osac describe securitygroup <securitygroup-name-or-id>
```

This displays the security group's state and a table of all ingress and egress rules.

---

## Next steps

With networking in place, you can create compute instances that attach to your Subnets
(and optionally SecurityGroups). See [Creating a VM with OSAC](computeinstance-guide.md).
