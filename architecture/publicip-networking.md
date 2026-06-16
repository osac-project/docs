# PublicIP Networking

This document describes how OSAC provides publicly routable IP addresses for
ComputeInstances (KubeVirt VMs). It covers the problem being solved, the
resource model, the traffic flow, and the system components involved.

For a step-by-step usage guide, see
[Assigning a Public IP to a ComputeInstance](../guides/developer/publicip-guide.md).

## Contents

- [Problem: VMs are isolated by default](#problem-vms-are-isolated-by-default)
- [Solution: PublicIP + MetalLB L2](#solution-publicip--metallb-l2)
- [Resource model](#resource-model)
- [Lifecycle](#lifecycle)
- [Traffic flow](#traffic-flow)
- [System architecture](#system-architecture)

---

## Problem: VMs are isolated by default

KubeVirt VMs run as pods on an OVN overlay network. They receive pod IPs (for
example, 10.128.x.x) that are internal to the cluster. Masquerade networking
allows VMs to initiate outbound connections, but external clients have no
route to the pod network and cannot reach the VM:

```
  External Client                    OpenShift Cluster
                                 +---------------------+
       |                         |                     |
       |  Can I reach            |  Pod Network        |
       |  10.128.0.81:22?        |  (OVN overlay)      |
       |                         |  10.128.0.0/14      |
       X--- NO ----------X       |                     |
                                 |  +----------------+ |
       The pod network is an     |  | virt-launcher  | |
       overlay. No route exists  |  | pod            | |
       from the external         |  |  10.128.0.81   | |
       network.                  |  |                | |
                                 |  |  +-----------+ | |
                                 |  |  | KubeVirt  | | |
                                 |  |  | VM        | | |
                                 |  |  +-----------+ | |
                                 |  +----------------+ |
                                 +---------------------+
```

Use cases that require inbound connectivity (SSH access, hosting a web
service, accepting API calls) need a routable address on the physical network.

---

## Solution: PublicIP + MetalLB L2

OSAC provisions a MetalLB LoadBalancer Service that advertises a public IP
address on the physical network via ARP (Layer 2 mode). External traffic to
the public IP is routed through the Kubernetes Service to the VM pod:

```
  External Client                    OpenShift Cluster
                                 +---------------------+
       |                         |                     |
       |  ssh user@              |  MetalLB Speaker    |
       |  203.0.113.10           |  (L2 mode)          |
       |                         |                     |
       +-------- ARP ----------->|  "203.0.113.10      |
       |  "Who has               |   is at MAC         |
       |   203.0.113.10?"        |   aa:bb:cc:dd"      |
       |                         |                     |
       +--- TCP SYN ------------>|  LoadBalancer Svc   |
       |                         |  203.0.113.10       |
       |                         |       |             |
       |                         |       | kube-proxy  |
       |                         |       | DNAT        |
       |                         |       v             |
       |                         |  +---------------+  |
       |                         |  | virt-launcher  | |
       |                         |  | pod            | |
       |                         |  |  10.128.0.81   | |
       |                         |  |                | |
       |                         |  |  +-----------+ | |
       |  SSH Connected!         |  |  | KubeVirt  | | |
       |<--- TCP SYN-ACK --------|  |  | VM        | | |
       |                         |  |  +-----------+ | |
       |                         |  +---------------+  |
       |                         +---------------------+
```

The MetalLB Speaker responds to ARP requests for the public IP, claiming it
on the local network segment. Once ARP resolves, the Kubernetes Service
receives the traffic and kube-proxy DNATs it to the VM's pod IP.

---

## Resource model

PublicIP networking uses three resource types with a clear separation between
provider-managed infrastructure and tenant self-service:

```
  +-------------------------------------+
  | PublicIPPool                        |  <-- Cloud Provider Admin (private API)
  |   CIDRs: 203.0.113.0/28             |
  |   IP family: IPv4                   |
  |   State: Ready                      |
  |   Available: 12                     |
  +-------------------------------------+
                     |
                     |  tenant allocates from pool
                     v
  +-------------------------------------+
  | PublicIP                            |  <-- Tenant User (public API)
  |   Pool: <pool-name>                 |
  |   Address: 203.0.113.10             |
  |   State: Allocated                  |
  |   Attached: false                   |
  +-------------------------------------+
                     |
                     |  tenant binds to ComputeInstance
                     v
  +-------------------------------------+
  | PublicIPAttachment                  |  <-- Tenant User (public API)
  |   PublicIP: <publicip-name>         |
  |   Target: <computeinstance-name>    |
  |   State: Ready                      |
  +-------------------------------------+
```

### PublicIPPool

A range of routable IP addresses provisioned by the cloud provider.

- Created via the **private (admin) API** only.
- Defines one or more CIDR ranges and an IP family (IPv4 or IPv6). A single
  pool cannot mix address families.
- Tracks capacity: total addresses, currently allocated, and available.
- CIDRs and IP family are immutable after creation.
- Tenants can list and inspect pools via the public API but cannot create,
  modify, or delete them.

### PublicIP

A single routable address allocated from a PublicIPPool.

- Created by **tenants** via the public API.
- The pool assignment is required at creation time and immutable.
- Lifecycle: `Pending` (waiting for address allocation) then `Allocated`
  (address assigned and reserved).
- Independent of any ComputeInstance. A PublicIP persists until explicitly
  deleted, allowing tenants to reserve and reassign addresses.
- Cannot be deleted while attached. The attachment must be removed first.

### PublicIPAttachment

The binding between a PublicIP and a target resource (currently
ComputeInstance, with future support for other resource types).

- Created by **tenants** via the public API.
- Both the PublicIP reference and the target are immutable. To change the
  target, delete the attachment and create a new one.
- Lifecycle: `Pending` (attach in progress) then `Ready` (traffic routing).
- Deleting the attachment triggers the detach workflow but does not release
  the PublicIP.
- One-to-one binding: each PublicIP can have at most one attachment, and each
  ComputeInstance can have at most one PublicIPAttachment.

---

## Lifecycle

The full lifecycle spans three phases, each managed by the resource type
responsible for that stage:

```
  Phase 1: Pool Provisioning (Cloud Provider Admin)
  +----------------------------------------------------+
  |  PublicIPPool                                      |
  |    CIDRs: 203.0.113.0/28                           |
  |    State: Pending -> Ready                         |
  |                                                    |
  |  The system creates a MetalLB IPAddressPool on     |
  |  the target cluster and configures L2              |
  |  advertisement for the CIDR ranges.                |
  +----------------------------------------------------+
       |
       |  Tenant allocates from pool
       v
  Phase 2: IP Allocation (Tenant User)
  +----------------------------------------------------+
  |  PublicIP                                          |
  |    Address: 203.0.113.10                           |
  |    State: Pending -> Allocated                     |
  |    Attached: false                                 |
  |                                                    |
  |  A "parking" LoadBalancer Service is created with  |
  |  an empty selector. This reserves the IP via ARP   |
  |  but does not route traffic to any workload.       |
  +----------------------------------------------------+
       |
       |  Tenant creates PublicIPAttachment
       v
  Phase 3: Attachment (Tenant User)
  +----------------------------------------------------+
  |  PublicIPAttachment                                |
  |    PublicIP: <publicip-name>                       |
  |    ComputeInstance: <computeinstance-name>         |
  |    State: Pending -> Ready                         |
  |                                                    |
  |  The system moves the Service into the             |
  |  ComputeInstance's namespace with a selector       |
  |  matching the VM pod. MetalLB advertises the IP    |
  |  and kube-proxy DNATs traffic to the pod.          |
  |                                                    |
  |  PublicIP.attached = true                          |
  +----------------------------------------------------+
```

### Detach and release

Detaching reverses Phase 3: the Service is moved back to a parking namespace
with an empty selector. The PublicIP remains `Allocated` and can be reattached
to a different ComputeInstance.

Releasing (deleting the PublicIP) reverses Phase 2: the parking Service is
removed and the address is returned to the pool's available capacity.

---

## Traffic flow

Once a PublicIPAttachment reaches `Ready` state, the packet path for an
inbound connection looks like this:

1. **External client** sends a packet to the public IP (e.g., 203.0.113.10).
2. **MetalLB Speaker** responds to ARP for that address, claiming it on the
   node's physical NIC.
3. **kube-proxy** matches the packet to the LoadBalancer Service and DNATs it
   to the VM's pod IP (e.g., 10.128.0.81).
4. **OVN overlay** delivers the packet to the virt-launcher pod.
5. **KubeVirt VM** receives the connection.

Return traffic follows the reverse path. From the external client's
perspective, the VM appears to have a stable routable address on the physical
network.

### Service swap pattern

The attach/detach mechanism uses a "service swap" pattern:

```
  ALLOCATED (parking)               ATTACHED (routing)

  parking namespace                  VM's namespace
  +------------------------+        +------------------------+
  | Service: parking-svc   |        | Service: ingress-svc   |
  | type: LoadBalancer     |        | type: LoadBalancer     |
  | loadBalancerIP:        |        | loadBalancerIP:        |
  |   203.0.113.10         |        |   203.0.113.10         |
  | selector: {}  (empty)  |  ==>   | selector:              |
  |                        |        |   vm.kubevirt.io/name: |
  | IP reserved via ARP    |        |     <vm-name>          |
  | but not routed.        |        |                        |
  +------------------------+        | Traffic routed to VM.  |
                                    +------------------------+

  Attach: delete parking-svc, create ingress-svc in VM namespace
  Detach: delete ingress-svc, recreate parking-svc
```

---

## System architecture

The PublicIP workflow involves several OSAC components working together:

```
                              Forward path (provisioning)
                 ────────────────────────────────────────────────>

  +-----------+  +-----------+  +------------+  +------------+  +------------+
  | osac CLI  |  | fulfill-  |  | fulfill-   |  | osac-      |  | AAP        |
  |           |->| ment-     |->| ment-      |->| operator   |->|            |
  | tenant or |  | service   |  | controller |  |            |  | Ansible    |
  | admin     |  |           |  |            |  | watches    |  | roles      |
  |           |  | gRPC/REST |  | reconciles |  | CRs,       |  |            |
  +-----------+  | PostgreSQL|  | API to CRs |  | launches   |  +------+-----+
                 +-----+-----+  +------------+  | AAP jobs   |         |
                       ^                         +------------+   configures
                       |                                          MetalLB Svcs
                 gRPC Signal                                      + IPPools
                 (status updates)                                       |
                       |                                          +-----+------+
                 +-----+------+      watches CR status            | MetalLB    |
                 | feedback-  |<----------------------------------| (L2 mode)  |
                 | controller |      on workload cluster          |            |
                 +------------+                                   +------------+

                 <────────────────────────────────────────────────
                              Feedback path (status reporting)
```

**Forward path (provisioning):**

1. **Tenant/admin** uses the osac CLI (or REST/gRPC API) to create PublicIP
   resources.
2. **fulfillment-service** persists the resource in PostgreSQL and streams
   events to the fulfillment-controller.
3. **fulfillment-controller** reconciles API resources into Kubernetes custom
   resources (CRs) on the target cluster.
4. **osac-operator** watches the CRs and launches Ansible Automation Platform
   (AAP) jobs for the actual network configuration.
5. **AAP roles** create and manage MetalLB IPAddressPools, L2Advertisements,
   and LoadBalancer Services on the target cluster.
6. **MetalLB** advertises the allocated IPs via ARP and works with kube-proxy
   to route traffic to the correct VM pod.

**Feedback path (status reporting):**

7. **feedback-controller** watches CR status on the workload cluster (phase
   transitions, allocated addresses) and reports changes back to the
   fulfillment-service via gRPC Signal RPCs, closing the control loop.
