# CUDN Localnet Networking Flow Diagrams

## Physical Topology

Two SNO clusters connected to a spine-leaf fabric. Each node has a
single data NIC on br-ex, connected to a leaf switch as a trunk port.

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'lineColor': '#ff9933'}, 'flowchart': {'defaultRenderer': 'elk'}}}%%
graph TB
    internet((Internet))
    nat[NAT Gateway<br/>SNAT / DNAT]

    subgraph spine [Spine Layer]
        spine1[Spine 1]
        spine2[Spine 2]
    end

    subgraph leaf [Leaf Layer]
        leaf1[Leaf 1]
        leaf2[Leaf 2]
    end

    subgraph sno1 [SNO Cluster 1]
        brex1[br-ex<br/>trunk port]
        subgraph ovn1 [OVN br-int]
            vm_a1[vm-alpha<br/>10.0.1.6<br/>VLAN 3]
            vm_b1[vm-beta<br/>10.0.3.2<br/>VLAN 5]
        end
    end

    subgraph sno2 [SNO Cluster 2]
        brex2[br-ex<br/>trunk port]
        subgraph ovn2 [OVN br-int]
            vm_a2[vm-alpha<br/>10.0.1.128<br/>VLAN 3]
            vm_b2[vm-beta<br/>10.0.3.128<br/>VLAN 5]
        end
    end

    internet --- nat
    nat --- spine1
    nat --- spine2
    spine1 --- leaf1
    spine1 --- leaf2
    spine2 --- leaf1
    spine2 --- leaf2
    leaf1 --- brex1
    leaf2 --- brex2
    brex1 --- ovn1
    brex2 --- ovn2
```

## Single Switch Port: VLAN Multiplexing on br-ex

A single physical NIC carries all traffic. OVN tags VM frames with the
appropriate VLAN at the br-int localnet port. br-ex passes everything
transparently as a trunk.

```mermaid
%%{init: {'theme': 'dark'}}%%
graph LR
    subgraph node [OCP Node]
        subgraph brint [br-int]
            vm_a[VM Alpha<br/>10.0.1.6]
            vm_b[VM Beta<br/>10.0.3.2]
            lp3[localnet port<br/>VLAN 3 tag]
            lp5[localnet port<br/>VLAN 5 tag]
            mgmt[Node mgmt<br/>untagged]
        end
        brex[br-ex<br/>transparent trunk]
        nic[Physical NIC]
    end
    switch[Leaf Switch<br/>trunk port]

    vm_a --> lp3
    vm_b --> lp5
    lp3 -->|"tagged VLAN 3"| brex
    lp5 -->|"tagged VLAN 5"| brex
    mgmt -->|"untagged"| brex
    brex --> nic
    nic -->|"native + VLAN 3 + VLAN 5"| switch
```

## Flow 1: Same Virtual Network, Same Cluster (L2)

Traffic stays within OVN on the same node — never hits the fabric.
Both VMs are on the same OVN logical switch.

```mermaid
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant A as vm-alpha<br/>10.0.1.6
    participant OVN as OVN br-int<br/>(same node)
    participant B as vm-alpha-2<br/>10.0.1.9

    A->>OVN: Ethernet frame (untagged inside OVN)
    Note over OVN: Same logical switch<br/>L2 forwarding<br/>No fabric involved
    OVN->>B: Ethernet frame
    Note over A,B: TTL=64 (no routing hop)
```

## Flow 2: Same Virtual Network, Cross-Cluster (L2)

Both VMs are on the same VLAN. Traffic exits br-ex tagged, traverses
the fabric as a VXLAN/EVPN segment between leaf switches, and enters
the remote node's br-ex where OVN strips the tag.

```mermaid
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant A as vm-alpha C1<br/>10.0.1.6
    participant OVN1 as OVN br-int<br/>Cluster 1
    participant BREX1 as br-ex<br/>Cluster 1
    participant L1 as Leaf 1
    participant SP as Spine
    participant L2 as Leaf 2
    participant BREX2 as br-ex<br/>Cluster 2
    participant OVN2 as OVN br-int<br/>Cluster 2
    participant B as vm-alpha C2<br/>10.0.1.128

    A->>OVN1: Ethernet frame
    OVN1->>BREX1: Push VLAN 3 tag
    BREX1->>L1: 802.1Q tagged (VLAN 3)
    Note over L1: VLAN 3 → VNI mapping
    L1->>SP: VXLAN encap (fabric overlay)
    SP->>L2: VXLAN forwarding
    Note over L2: VNI → VLAN 3 mapping
    L2->>BREX2: 802.1Q tagged (VLAN 3)
    BREX2->>OVN2: Strip VLAN 3 tag
    OVN2->>B: Ethernet frame

    Note over A,B: TTL=64 (L2 switching, no routing hop)
```

## Flow 3: Different Virtual Networks, Same VPC/VRF (L3 Routed)

VMs are on different VLANs but share a VPC/VRF. The fabric leaf gateway
performs inter-VLAN routing. Requires explicit routes on VMs.

```mermaid
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant A as vm-alpha<br/>10.0.1.6
    participant OVN as OVN br-int
    participant BREX as br-ex
    participant LEAF as Leaf Switch
    participant GW_A as Gateway<br/>10.0.1.1
    participant GW_B as Gateway<br/>10.0.3.1
    participant B as vm-beta<br/>10.0.3.2

    Note over A: ip route: 10.0.3.0/24<br/>via 10.0.1.1 dev enp2s0
    A->>OVN: dst=10.0.3.2, src=10.0.1.6
    OVN->>BREX: Push VLAN 3 tag
    BREX->>LEAF: 802.1Q tagged (VLAN 3)
    LEAF->>GW_A: Arrives at anycast GW 10.0.1.1
    Note over GW_A,GW_B: Inter-VLAN routing<br/>within VPC/VRF<br/>TTL decremented 64→63
    GW_B->>LEAF: Route to 10.0.3.0/24
    LEAF->>BREX: 802.1Q tagged (VLAN 5)
    BREX->>OVN: Strip VLAN 5 tag
    OVN->>B: Ethernet frame

    Note over A,B: TTL=63 (one L3 hop through fabric gateway)
```

## Flow 4: Different Virtual Networks, Different VPCs/VRFs (Isolated)

VMs are on different VLANs in different VRFs. The fabric has no routing
path between the VRFs. Traffic is dropped.

```mermaid
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant A as vm-alpha<br/>10.0.1.6<br/>VPC-1 / VLAN 3
    participant OVN as OVN br-int
    participant BREX as br-ex
    participant LEAF as Leaf Switch
    participant B as vm-beta<br/>10.0.3.2<br/>VPC-2 / VLAN 5

    A->>OVN: dst=10.0.3.2
    Note over OVN: No route to 10.0.3.0/24<br/>on fabric interface
    Note over A: Traffic goes to pod network<br/>default route (enp1s0)<br/>Never reaches fabric
    Note over A,B: RESULT: No connectivity<br/>VRF isolation enforced
```

## Flow 5: SNAT via Fabric

VM traffic exits through the fabric to the NAT gateway when a default
route via the fabric gateway is configured.

```mermaid
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant VM as vm-alpha<br/>10.0.1.6
    participant OVN as OVN br-int
    participant BREX as br-ex
    participant LEAF as Leaf Switch
    participant GW as Gateway<br/>10.0.1.1
    participant NAT as NAT Gateway<br/>SNAT
    participant INET as Internet

    Note over VM: default via 10.0.1.1<br/>dev enp2s0 metric 50
    VM->>OVN: dst=8.8.8.8, src=10.0.1.6
    OVN->>BREX: Push VLAN 3 tag
    BREX->>LEAF: 802.1Q tagged (VLAN 3)
    LEAF->>GW: Arrives at anycast GW
    GW->>NAT: Default route to NAT gateway
    Note over NAT: SNAT: 10.0.1.6 → public IP
    NAT->>INET: src=131.153.207.174

    Note over VM,INET: Without fabric default route,<br/>traffic uses pod network instead<br/>(different public IP)
```

## OVN Internal Architecture

How OVN bridges localnet traffic from a VM to the physical wire.

```mermaid
%%{init: {'theme': 'dark'}}%%
graph TB
    subgraph vm [VM Guest]
        eth0[enp1s0<br/>masquerade<br/>10.0.2.2]
        eth1[enp2s0<br/>fabric<br/>10.0.1.6]
    end

    subgraph launcher [virt-launcher pod]
        tap0[tap device]
        bridge[Linux bridge]
    end

    subgraph brint [OVN br-int]
        vif[VIF port<br/>VM logical port]
        ls[Logical Switch<br/>tenant-alpha]
        lp[Localnet Port<br/>type=localnet<br/>network_name=localnet-physnet<br/>tag=3]
        patch1[patch port<br/>br-int side]
    end

    subgraph brex [br-ex]
        patch2[patch port<br/>br-ex side]
        uplink[Physical NIC<br/>uplink port]
    end

    wire[Physical Wire<br/>to Leaf Switch]

    eth1 --> tap0
    tap0 --> bridge
    bridge --> vif
    vif --> ls
    ls --> lp
    Note1[/"OVN pushes VLAN 3 tag<br/>at localnet port boundary"/]
    lp --> patch1
    patch1 -->|"VLAN 3 tagged"| patch2
    patch2 --> uplink
    uplink -->|"trunk: native + VLAN 3 + VLAN 5"| wire

    style Note1 fill:none,stroke:none,color:#ff9933
```

## Multi-Cluster IP Partitioning

How `excludeSubnets` prevents IP collisions across clusters sharing
the same virtual network.

```mermaid
%%{init: {'theme': 'dark'}}%%
graph TB
    subgraph subnet ["Virtual Network Alpha — 10.0.1.0/24"]
        subgraph reserved ["Reserved"]
            net["10.0.1.0 (network)"]
            gw["10.0.1.1 (gateway)"]
            bcast["10.0.1.255 (broadcast)"]
        end

        subgraph c1 ["Cluster 1 — OVN IPAM"]
            c1range["10.0.1.2 – 10.0.1.127"]
            c1vms["vm-alpha: 10.0.1.6"]
        end

        subgraph c2 ["Cluster 2 — OVN IPAM"]
            c2range["10.0.1.128 – 10.0.1.254"]
            c2vms["vm-alpha: 10.0.1.128"]
        end
    end

    c1 ---|"excludeSubnets:<br/>10.0.1.128/25"| c2
    c2 ---|"excludeSubnets:<br/>10.0.1.0/25"| c1
```
