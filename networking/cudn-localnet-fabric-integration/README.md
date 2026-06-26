# CUDN Localnet: Connecting KubeVirt VMs to the Physical Fabric via VLANs

## Problem

You have a managed spine-leaf fabric (e.g., Netris, OpenStack Neutron,
or similar). OCP cluster nodes connect to leaf switches. VMs run on
these nodes via KubeVirt. You want VMs on different nodes (and different
OCP clusters) to share the same L2 segment through the fabric — without
deploying FRR-K8s, configuring BGP peering, or setting up VXLAN/EVPN on
the OCP side.

**Goal:** Use CUDN Localnet topology and VLAN tagging to bridge KubeVirt
VM traffic directly onto a fabric virtual network. A single switch port
per server carries both br-ex (node management) and localnet (VM tenant)
traffic in tagged mode.

See [diagrams.md](diagrams.md) for detailed Mermaid diagrams of all
networking flows.

## How It Works

Instead of tunneling (EVPN/VXLAN from the OCP node), localnet uses raw
802.1Q VLAN tags on the physical wire:

1. The fabric manager creates a virtual network with a VLAN ID
2. The server's switch port trunks that VLAN alongside the existing
   management traffic
3. On OCP, a CUDN with Localnet topology maps to br-ex via
   bridge-mappings, specifying the same VLAN ID
4. OVN inserts/strips the VLAN tag at the br-ex port level
5. Tagged frames exit the physical NIC → reach the leaf switch →
   the leaf maps the VLAN to the fabric overlay (e.g., VXLAN/EVPN
   between switches) → delivered to other ports on the same network

VMs appear on the fabric virtual network as if they were bare-metal
servers on a tagged switch port. The fabric handles all inter-switch
forwarding via its native control plane. The OCP cluster doesn't need
to know anything about the fabric overlay.

The physical NIC on br-ex carries everything as a trunk. The leaf switch
differentiates traffic by VLAN tag — not by source IP. When multiple
virtual networks are assigned to the same port, each gets its own tagged
VLAN. A single switch port can carry the management network (native/
untagged) plus many tenant VLANs simultaneously.

## Prerequisites

### Fabric Side

- Managed spine-leaf fabric with a controller (e.g., Netris, OpenStack)
- NAT gateway (e.g., SoftGate, Neutron router) for internet access
- OCP nodes' physical NICs connected to leaf switch ports
- The ability to create virtual networks with:
  - A VLAN ID (auto-assigned or manual)
  - An anycast/distributed gateway on the subnet
  - Switch port trunk membership
  - DHCP **disabled** (OVN IPAM handles IP assignment)

### OpenShift Side

- OpenShift 4.19+ (CUDN Localnet is GA in 4.19)
- OVN-Kubernetes as the CNI
- NMState Operator installed (for bridge-mappings via NNCP)
- KubeVirt / OpenShift Virtualization for running VMs

### Important: Subnet Selection

**Do NOT use 10.0.2.0/24** for virtual network subnets. KubeVirt's
masquerade interface always assigns 10.0.2.2/24 to VMs internally on
the default pod network interface (enp1s0). If a virtual network uses
the same subnet, both VM interfaces get IPs in 10.0.2.0/24, breaking
routing on the fabric interface.

Safe subnet choices: 10.0.1.0/24, 10.0.3.0/24, 10.0.4.0/24,
172.16.x.0/24, etc.

## Step-by-Step Setup

### Step 1: Install NMState Operator (if not installed)

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nmstate
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nmstate
  namespace: openshift-nmstate
spec:
  targetNamespaces:
    - openshift-nmstate
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for operator, then create NMState instance:
oc apply -f - <<'EOF'
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
EOF
```

### Step 2: Create Fabric Virtual Networks

Using your fabric controller, create a virtual network for each tenant
segment. Each virtual network needs:

- **Subnet and gateway**: e.g., 10.0.1.0/24 with gateway 10.0.1.1
- **VLAN ID**: auto-assigned or manually chosen
- **Switch port membership**: add the OCP node switch ports to the
  virtual network. The fabric configures those ports as trunks.
- **DHCP disabled**: OVN IPAM on the OCP side assigns IPs to VMs

Record the VLAN ID assigned to each virtual network — you'll need it
for the CUDN configuration.

### Step 3: Configure Bridge-Mappings (NNCP)

Add a bridge-mapping that tells OVN-Kubernetes: "the physical network
named `localnet-physnet` is reachable via the `br-ex` bridge."

Apply on **every cluster** that will use localnet:

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: localnet-bridge-mapping
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    ovn:
      bridge-mappings:
        - localnet: localnet-physnet
          bridge: br-ex
          state: present
```

This is additive — it appends `localnet-physnet:br-ex` alongside the
existing `physnet:br-ex` mapping. It does NOT modify br-ex itself.

**Verification:**

```bash
oc get nncp localnet-bridge-mapping
# STATUS: Available

oc get nns <node-name> -o json | \
  jq '.status.currentState.ovn."bridge-mappings"'
# Should show both:
#   {"bridge": "br-ex", "localnet": "physnet"}
#   {"bridge": "br-ex", "localnet": "localnet-physnet"}
```

### Step 4: Create Namespace and CUDN

Create a namespace and CUDN for each virtual network. The CUDN VLAN ID
must match the fabric virtual network's VLAN.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-alpha
  labels:
    network: tenant-alpha
---
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: tenant-alpha
spec:
  namespaceSelector:
    matchLabels:
      network: tenant-alpha
  network:
    topology: Localnet
    localnet:
      role: Secondary
      physicalNetworkName: localnet-physnet
      vlan:
        mode: Access
        access:
          id: 3                       # Must match fabric VLAN ID
      subnets:
        - "10.0.1.0/24"              # Must match fabric subnet
      excludeSubnets:
        - "10.0.1.0/32"              # Network address
        - "10.0.1.1/32"              # Fabric gateway
        - "10.0.1.255/32"            # Broadcast address
      ipam:
        mode: Enabled
        lifecycle: Persistent
```

**For multi-cluster deployments**, partition the IP range using
`excludeSubnets` to prevent OVN IPAM collisions:

- **Cluster 1:** exclude `10.0.1.128/25` → allocates from 10.0.1.2–127
- **Cluster 2:** exclude `10.0.1.0/25` → allocates from 10.0.1.128–254

**Verification:**

```bash
oc get clusteruserdefinednetwork tenant-alpha -o yaml
# status.conditions should show NetworkCreated: True
# with NAD created in the matching namespace
```

### Step 5: Create VMs

Each VM gets two interfaces:
- **default**: pod network (masquerade) — Kubernetes health probes
- **fabric**: the localnet CUDN — provides fabric connectivity

Include an SSH public key in cloud-init for access via jump pods:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-alpha
  namespace: tenant-alpha
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
            - name: fabric
              bridge: {}
          disks:
            - name: rootdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
        resources:
          requests:
            memory: 2Gi
      networks:
        - name: default
          pod: {}
        - name: fabric
          multus:
            networkName: tenant-alpha
      volumes:
        - name: rootdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:latest
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              user: fedora
              password: changeme
              chpasswd:
                expire: false
              ssh_pwauth: true
              ssh_authorized_keys:
                - <your-ssh-public-key>
```

**Verification:**

```bash
oc get vmi -n tenant-alpha
# Should show Running

oc get vmi vm-alpha -n tenant-alpha \
  -o jsonpath='{range .status.interfaces[*]}{.name}: {.ipAddress}{"\n"}{end}'
# default: 10.128.x.x
# fabric: 10.0.1.x
```

### Step 6: Create Jump Pods for SSH Access

`virtctl ssh` connects via the pod network, which breaks if you modify
the VM's default route. Use a jump pod on the localnet instead:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jump
  namespace: tenant-alpha
  annotations:
    k8s.v1.cni.cncf.io/networks: tenant-alpha
spec:
  containers:
  - name: tools
    image: registry.redhat.io/rhel9/support-tools:latest
    command: ["sleep", "infinity"]
```

Copy your SSH key and connect to VMs:

```bash
oc cp ~/.ssh/id_ed25519 tenant-alpha/jump:/tmp/id_ed25519
oc exec -n tenant-alpha jump -- chmod 600 /tmp/id_ed25519

# SSH into VM via fabric IP:
oc exec -n tenant-alpha jump -- \
  ssh -i /tmp/id_ed25519 \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  fedora@10.0.1.6 "ip addr show enp2s0"
```

### Step 7: Fabric SNAT (Optional)

Create an SNAT rule in your fabric controller for each virtual network
to give VMs internet access through the fabric (not through the pod
network).

**Verifying SNAT uses the fabric path (not pod network):**

VMs have two default routes — the pod network (lower metric wins by
default) and potentially the fabric. To test SNAT, add a default route
via the fabric gateway:

```bash
# From jump pod, SSH into VM:
oc exec -n tenant-alpha jump -- ssh -i /tmp/id_ed25519 \
  -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  fedora@10.0.1.6 "
  sudo ip route add default via 10.0.1.1 dev enp2s0 metric 50
  curl -s ifconfig.me     # Shows fabric SNAT public IP
  sudo ip route del default via 10.0.1.1 dev enp2s0 metric 50
  curl -s ifconfig.me     # Shows pod network public IP (different)
"
```

Two different public IPs confirm the two traffic paths.

**Warning:** Adding a lower-metric default route via the fabric breaks
`virtctl ssh` (return traffic goes out the wrong interface). Always use
the jump pod approach when the fabric default route is active.

## Connectivity Model

### How traffic flows between virtual networks depends on VPC/VRF placement

Fabric virtual networks can be in the **same VPC/VRF** or **different
VPCs/VRFs**. This determines L3 connectivity:

| Virtual network placement | L2 (same subnet) | L3 (cross-subnet) |
|---|---|---|
| Same virtual network | Direct switching (TTL=64) | N/A (same subnet) |
| Different virtual networks, same VPC/VRF | No (different VLAN/broadcast domain) | Routed via fabric gateway (TTL=63) — requires routes on VMs |
| Different virtual networks, different VPCs/VRFs | No | No (VRF isolation) |

### L3 routing between virtual networks in the same VPC/VRF

When two virtual networks share a VPC/VRF, the fabric leaf switches can
route between them (inter-VLAN routing via anycast/distributed gateways).
However, VMs don't automatically know how to reach other subnets — their
fabric interface only has a connected route for its own subnet.

To enable L3 connectivity, add routes on each VM:

```bash
# On a VM in virtual network Alpha (10.0.1.0/24), to reach Beta (10.0.3.0/24):
sudo ip route add 10.0.3.0/24 via 10.0.1.1 dev enp2s0

# On a VM in virtual network Beta, to reach Alpha:
sudo ip route add 10.0.1.0/24 via 10.0.3.1 dev enp2s0
```

Traffic path: VM-Alpha → enp2s0 → VLAN 3 → leaf gateway →
inter-VLAN routing within VRF → VLAN 5 → VM-Beta. TTL decrements
from 64 to 63, confirming the L3 hop.

**Note:** Without explicit routes on the VMs, cross-subnet traffic goes
through the pod network default route instead of the fabric, and never
reaches the fabric gateway.

## Verified Test Results

The following tests were performed with 2 SNO clusters on a managed
fabric, 2 virtual networks (Alpha on VLAN 3, Beta on VLAN 5), 4 VMs
total.

### Test Environment

```
Cluster 1 (OCP 4.22):
  vm-alpha  → Virtual Network Alpha (VLAN 3)  → fabric IP: 10.0.1.6
  vm-beta   → Virtual Network Beta  (VLAN 5)  → fabric IP: 10.0.3.2

Cluster 2 (OCP 4.21):
  vm-alpha  → Virtual Network Alpha (VLAN 3)  → fabric IP: 10.0.1.128
  vm-beta   → Virtual Network Beta  (VLAN 5)  → fabric IP: 10.0.3.128
```

### Test 1: Same virtual network, same cluster (L2)

Two VMs on Alpha within Cluster 1:

```
vm-alpha (10.0.1.6) → vm-alpha-2 (10.0.1.9)
Result: PING OK, TTL=64 (L2 switching, same broadcast domain)
ARP table shows peer MAC on enp2s0
```

### Test 2: Same virtual network, cross-cluster (L2)

VMs on Alpha across Cluster 1 and Cluster 2:

```
vm-alpha C1 (10.0.1.6) → vm-alpha C2 (10.0.1.128)
Result: PING OK, TTL=64, ~4ms
Traffic path: br-ex (VLAN 3) → Leaf-1 → Spine → Leaf-2 → br-ex → VM
```

VMs on Beta across clusters:

```
vm-beta C2 (10.0.3.128) → vm-beta C1 (10.0.3.2)
Result: PING OK, TTL=64, ~4ms
```

### Test 3: Different virtual networks, same VPC/VRF (L3 routed)

Alpha and Beta in the same VPC/VRF, with routes added on VMs:

```
vm-alpha C1 (10.0.1.6) → vm-beta C1 (10.0.3.2)
Route added: sudo ip route add 10.0.3.0/24 via 10.0.1.1 dev enp2s0
           + sudo ip route add 10.0.1.0/24 via 10.0.3.1 dev enp2s0 (on beta)
Result: PING OK, TTL=63 (routed through fabric leaf gateway)
```

Cross-gateway reachability (10.0.3.1 from an Alpha VM) confirmed at
TTL=64 — proving inter-VLAN routing works within the VPC/VRF.

### Test 4: Different virtual networks, different VPCs/VRFs (isolated)

Alpha and Beta in separate VPCs/VRFs:

```
vm-alpha C1 (10.0.1.6) → vm-beta C1 (10.0.3.2):     FAIL
vm-alpha C1 (10.0.1.6) → vm-beta C2 (10.0.3.128):    FAIL
vm-beta C2 (10.0.3.128) → vm-alpha C1 (10.0.1.6):    FAIL
vm-beta C2 (10.0.3.128) → vm-alpha C2 (10.0.1.128):  FAIL
vm-alpha C1 → Beta gateway (10.0.3.1):                FAIL
vm-beta C1 → Alpha gateway (10.0.1.1):                FAIL
```

Complete isolation — no L2 or L3 path between VPCs/VRFs, not even to
each other's gateways.

### Test 5: SNAT via fabric

```
vm-alpha C1 (10.0.1.6):
  Via fabric (default route via 10.0.1.1, metric 50):
    curl ifconfig.me → 131.153.207.174 (fabric SNAT IP)
  Via pod network (default route via 10.0.2.1, metric 100):
    curl ifconfig.me → 131.153.207.172 (OVN masquerade IP)
```

Different public IPs confirm the two traffic paths.

### Full Results Matrix

| From | To | Scenario | Expected | Result |
|------|----|----------|----------|--------|
| vm-alpha C1 | vm-alpha C1-2 | Same vnet, same cluster | L2 OK (TTL=64) | **PASS** |
| vm-alpha C1 | vm-alpha C2 | Same vnet, cross-cluster | L2 OK (TTL=64) | **PASS** |
| vm-beta C2 | vm-beta C1 | Same vnet, cross-cluster | L2 OK (TTL=64) | **PASS** |
| vm-alpha C1 | vm-beta C1 | Diff vnet, same VPC, same cluster | L3 OK (TTL=63) with routes | **PASS** |
| vm-alpha C1 | vm-beta C1 | Diff vnet, diff VPC, same cluster | Isolated | **PASS** |
| vm-alpha C1 | vm-beta C2 | Diff vnet, diff VPC, cross-cluster | Isolated | **PASS** |
| vm-beta C2 | vm-alpha C1 | Diff vnet, diff VPC, cross-cluster | Isolated | **PASS** |
| vm-alpha C1 | Beta GW | Diff vnet, diff VPC | Isolated | **PASS** |
| vm-alpha C1 | Alpha GW | Same vnet | OK | **PASS** |
| vm-alpha C1 | Internet (fabric) | SNAT | Fabric public IP | **PASS** |

## Multi-Cluster Setup

### IP Range Partitioning

When multiple clusters share the same virtual network, each cluster's
OVN IPAM allocates from the same subnet independently. Use
`excludeSubnets` to partition the range and prevent collisions:

**Cluster 1** — lower half:

```yaml
excludeSubnets:
  - "10.0.1.0/32"       # Network
  - "10.0.1.1/32"       # Gateway
  - "10.0.1.128/25"     # Reserve upper half for Cluster 2
  - "10.0.1.255/32"     # Broadcast
```

**Cluster 2** — upper half:

```yaml
excludeSubnets:
  - "10.0.1.0/25"       # Reserve lower half for Cluster 1
  - "10.0.1.255/32"     # Broadcast
```

### Fabric Side

Each virtual network must include the switch ports of **all** clusters'
nodes. When adding a new cluster, update the virtual network's port
membership in the fabric controller. Both switch ports will then trunk
the virtual network's VLAN, and the leaf switches connect them via the
fabric overlay through the spine layer.

### CUDN Configuration is Immutable

CUDNs cannot be modified after creation. If you need to change the VLAN
ID, subnet, or excludeSubnets, delete the CUDN and recreate it. VMs
must be deleted first since the CUDN controller blocks deletion while
workloads are attached.

## Comparison: Localnet vs EVPN vs VRF-Lite

| | Localnet (this doc) | EVPN | VRF-Lite |
|---|---|---|---|
| **OCP version** | 4.19+ | 4.22+ (TechPreview) | 4.19+ |
| **Fabric integration** | VLAN tags on wire | VXLAN tunnel from node | BGP route exchange |
| **FRR-K8s required** | No | Yes | Yes |
| **BGP peering** | No (fabric-only) | Node ↔ Leaf EVPN | Node ↔ Router VRF |
| **VTEP CRs** | No | Yes | No |
| **VM visibility** | Fabric sees VLAN, not individual MACs | Fabric sees VM MAC+IP via EVPN Type 2 | Fabric sees pod subnet via BGP |
| **NAT** | Via fabric (subnet-level SNAT/DNAT) | Via fabric (per-VM DNAT possible) | Via external router |
| **VM network role** | Secondary | Primary | Primary |
| **L3 cross-subnet** | Manual routes on VMs | Automatic (OVN routing) | Automatic (BGP routing) |
| **Complexity** | Low | High | Medium |
| **BM + VM same segment** | Yes (same VLAN) | Yes (same VNI/EVPN) | No (separate routing) |

### When to Use Which

**Localnet**: Simplest option. Use when you need VMs on a fabric
segment without any control-plane integration on the OCP side. Good
for tenant isolation via VLANs, external connectivity, and
mixed BM+VM environments where per-VM fabric visibility isn't needed.
The secondary network role means VMs need explicit routes for
cross-subnet traffic.

**EVPN**: Most capable. Use when you need per-VM MAC/IP visibility in
the fabric routing table, direct DNAT to individual VMs, or
integration with EVPN-based network services. The primary network role
means no routing complications. Requires OCP 4.22+ and FRR-K8s.

**VRF-Lite**: Use when you need routed (L3) connectivity between OCP
pods/VMs and an external router, with per-tenant VRF isolation.
Requires FRR-K8s and per-tenant VLAN+VRF configuration on both the
cluster and the router.

## Troubleshooting

### Bridge-mappings not applied

```bash
oc get nncp localnet-bridge-mapping -o yaml
# status.conditions should show Available: True

oc debug node/<node-name> -- chroot /host \
  ovs-vsctl get open_vswitch . external_ids:ovn-bridge-mappings
# Should contain "localnet-physnet:br-ex"
```

### CUDN not ready

```bash
oc get clusteruserdefinednetwork <name> -o yaml
# Check status.conditions for errors
# Common issue: physicalNetworkName doesn't match any bridge-mapping
```

### VM has no IP on fabric interface

```bash
# Check IPAM and subnets:
oc get clusteruserdefinednetwork <name> \
  -o jsonpath='{.spec.network.localnet.ipam}'

# Check namespace label matches CUDN selector:
oc get ns <name> --show-labels

# Check generated NAD exists:
oc get net-attach-def -n <namespace>

# Check VM interfaces via guest agent:
oc exec -n <ns> <virt-launcher-pod> -c compute -- \
  virsh domifaddr <domain> --source agent
```

### Both VM interfaces show the same IP (subnet collision)

If enp1s0 (masquerade) and enp2s0 (fabric) both show the same
IP/subnet, the virtual network subnet collides with KubeVirt's
masquerade network (10.0.2.0/24). Change the virtual network to a
different subnet and recreate the CUDN.

### VMs can't reach each other across clusters

```bash
# Verify VLAN tag on the node:
oc debug node/<node-name> -- chroot /host \
  ovs-ofctl dump-flows br-ex | grep "dl_vlan=<vlan-id>"

# Verify the fabric virtual network includes both clusters' switch ports

# Verify both clusters use the same VLAN ID in their CUDNs
```

### L3 cross-subnet traffic fails (same VPC/VRF)

VMs need explicit routes to reach other virtual network subnets via the
fabric gateway. Without routes, traffic goes through the pod network
default route and never reaches the fabric.

```bash
# Inside the VM:
ip route show
# Should show: 10.0.3.0/24 via 10.0.1.1 dev enp2s0

# If missing, add:
sudo ip route add 10.0.3.0/24 via 10.0.1.1 dev enp2s0
```

Also add the return route on the destination VM.

### virtctl ssh times out

If you added a default route via the fabric gateway (metric 50), return
SSH traffic goes out the fabric interface instead of the pod network,
breaking the `virtctl ssh` connection. Use jump pods instead:

```bash
oc exec -n <ns> jump -- ssh -i /tmp/id_ed25519 \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  fedora@<fabric-ip> "<command>"
```

## Key Configuration Parameters

| Parameter | Where | Purpose |
|-----------|-------|---------|
| Virtual network VLAN ID | Fabric controller | VLAN for the fabric segment |
| Virtual network ports | Fabric controller | Switch ports that trunk this VLAN |
| Virtual network gateway | Fabric controller | Anycast/distributed gateway IP |
| Virtual network VPC/VRF | Fabric controller | Determines L3 routing scope |
| Virtual network DHCP | Fabric controller | Must be **disabled** (OVN IPAM handles IPs) |
| Bridge-mapping name | NNCP `localnet` | Physical network name for OVN |
| Bridge-mapping bridge | NNCP `bridge` | OVS bridge (`br-ex` for single-NIC) |
| CUDN physicalNetworkName | CUDN spec | Must match NNCP `localnet` name |
| CUDN VLAN ID | CUDN `vlan.access.id` | Must match fabric VLAN ID |
| CUDN subnets | CUDN spec | Must match fabric virtual network subnet |
| CUDN excludeSubnets | CUDN spec | Reserve gateway IP + IP ranges for other clusters |

## Known Issues

### Same-Node Connectivity (OCPBUGS-43004)

In OCP versions prior to 4.19, VMs on a localnet secondary network
could not communicate with pods on the default network when both were
on the **same node**. Fixed in OCP 4.19+. If running 4.18, check for
the backport in 4.18.14+.

### NNCP Bridge-Mapping Destructive Bug (OCPBUGS-18869)

In OCP 4.14, applying an NNCP with `ovn.bridge-mappings` wiped ALL
existing OVS `external_ids` on each node. Fixed in OCP 4.15+.

### Upgrade Connectivity Loss (OCPBUGS-66994)

During cluster upgrades (4.17 to 4.18), VMs on localnet can lose
connectivity. Workaround: restart the ovnkube-node pod or live-migrate
the affected VM.

### NNCP Verification Errors with Active VMs

Modifying an NNCP while VMs are actively using localnet networks can
cause NMState VerificationError. Apply the NNCP before creating VMs,
or drain VMs first.

## Limitations

- **Secondary only**: Localnet CUDN is always a secondary network.
  VMs have two interfaces — the pod network (masquerade) for
  Kubernetes health probes, and the fabric interface for tenant
  traffic. This dual-interface setup means VMs need explicit routes
  for cross-subnet traffic via the fabric, and care must be taken
  with default route selection.

- **No per-VM fabric visibility**: Unlike EVPN, the fabric doesn't
  learn individual VM MAC addresses via control plane. MAC learning
  happens via data-plane flooding within the virtual network.

- **VLAN scale**: Each virtual network consumes a VLAN ID (1-4094).
  The EVPN approach uses 24-bit VNIs (16M segments) and doesn't have
  this limit.

- **IP coordination**: Cross-cluster deployments require manual IP
  range partitioning via `excludeSubnets`. No cross-cluster IPAM.

- **Immutable CUDN**: Cannot be modified after creation. Changes
  require recreating the CUDN and reattaching VMs.

- **Same-node traffic**: VM-to-VM traffic on the same node stays
  within OVN and never hits the fabric. Fabric ACLs don't apply.

- **Masquerade subnet collision**: Virtual network subnets must not
  overlap with 10.0.2.0/24 (KubeVirt masquerade internal subnet).
