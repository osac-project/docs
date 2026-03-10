# **C-UDN with Provider Network Integration**

## **Overview & Architecture**

This lab demonstrates how to implement a **ClusterUserDefinedNetwork (C-UDN)**
integrated with a Managed Service Provider (MSP) network fabric using **BGP**
and **VRF-Lite**.

- **C-UDN Isolation:** A Kubernetes namespace (or group of namespaces) is
  assigned to a C-UDN. This allows pods to receive IP addresses outside the
  cluster's primary Pod CIDR, providing complete network isolation at the
  workload level.
- **Provider Network Integration:** The OpenShift (OCP) cluster nodes peer with
  the MSP network fabric using BGP.
- **Architecture Decision:** We are using **VRF-Lite**. We provision a dedicated
  VLAN for each C-UDN and peer to the fabric over this VLAN interface. This
  preserves C-UDN isolation end-to-end within the customer network.

## **References**

- **Red Hat Documentation:** [Advertising Pod IPs from a User-Defined Network
  over BGP with
  VPN](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#advertising-pod-ips-from-a-user-defined-network-over-bgp-with-vpn_about-route-advertisements)\
- **Git repo - <https://github.com/eranco74/ocp-cudn-vrflite-lab>**

**Topology:**

![Topology](topology.png)

## **Step 1: Cluster Provisioning & Operator Preparation**

Before configuring individual nodes or namespaces, we must provision OpenShift
cluster and enable the required routing and OVN-Kubernetes features.

### **1.1 Provision the Cluster using dev-scripts**

        # Clone the dev-scripts repository
        git clone https://github.com/openshift-metal3/dev-scripts.git
        cd dev-scripts
        cat << EOF > config_root.sh
        export OPENSHIFT_RELEASE_STREAM=4.19
        export WORKING_DIR=/root/dell/disks/dev-scripts
        export CLUSTER_NAME=new
        export BASE_DOMAIN=redhat.com
        export IP_STACK=v4
        export BMC_DRIVER=redfish-virtualmedia
        export NUM_WORKERS=2
        export MASTER_VCPU=10
        export MASTER_MEMORY=36000
        export WORKER_DISK=200
        export MASTER_DISK=200
        export NUM_EXTRA_WORKERS=0
        export VM_EXTRADISKS=true
        export VM_EXTRADISKS_LIST=vda vdb
        export VM_EXTRADISKS_SIZE=200G
        EOF

        # Provision the cluster
        make

### **1.2 Enable Routing Via Host**

`oc patch network.operator.openshift.io/cluster --type=merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"routingViaHost":true}}}}}'`

### **1.3 Enable FRR Provider (Creates FRR CRDs)**

`oc patch network.operator.openshift.io/cluster --type=merge -p '{"spec":{"additionalRoutingCapabilities":{"providers":["FRR"]}}}'`

### **1.4 Enable Route Advertisements (Creates routeAdvertisements CRD)**

`oc patch network.operator cluster --type=merge --patch '{"spec":{     "defaultNetwork":{"ovnKubernetesConfig":{"routeAdvertisements":"Enabled"}}}}'`

### \*\*1.5 Enable Global IP Forwarding (Required to allow OVN-Kubernetes to forward

non-pod traffic across the host's secondary interfaces)\*\*

`oc patch Network.operator.openshift.io cluster --type=merge       -p='{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"ipForwarding":"Global"}}}}}'`

## **Step 2: Host Network Configuration (OpenShift Nodes)**

Configure the worker nodes to handle the VLAN and VRF traffic required for
VRF-Lite.

***Note: Perform the host network configuration on all worker nodes.***

Note: Transition this manual step to be managed declaratively by the OpenShift
Kubernetes NMState Operator using a NodeNetworkConfigurationPolicy for
production*.*

### **2.1 Create the VRF and VLAN Devices** SSH into your target host and identify

your primary interface (e.g., `enp1s0`). Run the following to create the VRF,
VLAN, and link them: `ip a`

### **2.2 Create the VLAN Device**

- Create the VLAN interface (links it to enp1s0 and sets the ID to 101):
  `sudo ip link add link enp1s0 name enp1s0.101 type vlan id 101`
- Bring the interface state UP: `sudo ip link set dev enp1s0.101 up`

*Verify the interfaces were created:* `ip a | grep enp1s0.101`

## Step 3: Tenant Namespace Configuration

To attach a namespace to the C-UDN, create a namespace and apply the necessary
labels to trigger OVN-Kubernetes.

### **3.1 Create the Tenant Namespace**

        cat << 'EOF' > tenant1_ns.yaml
        apiVersion: v1
        kind: Namespace
        metadata:
          name: tenant1
          labels:
            k8s.ovn.org/primary-user-defined-network: ""
            network: tenant1-net
        EOF

        oc apply -f tenant1_ns.yaml 

### **3.2 Create the ClusterUserDefinedNetwork (C-UDN)**

        cat << 'EOF' > cudn.yaml 
        apiVersion: k8s.ovn.org/v1
        kind: ClusterUserDefinedNetwork
        metadata:
          name: tenant1-net
          labels:
            export: "true"
        spec:
          namespaceSelector:
            matchLabels:
              network: tenant1-net
          network:
            topology: Layer2 # use single subnetwork for all connected VMs / pods
            layer2: 
              role: Primary # network interface inside the pod will act as primary interface
              subnets: [10.0.1.0/24] 
              ipam: {lifecycle: Persistent} # ensure VMs will have their IP retained on live-migration, Stop or Restart
        EOF

        oc apply -f cudn.yaml

### **3.3 Check a vrf device was created on all worker nodes**

        [core@worker-1 ~]$ ip vrf show
        Name              Table
        -----------------------
        tenant1-net       1008
        tenant2-net       1009

### **3.4 Link the VRF to the VLAN**

`sudo ip link set dev enp1s0.101 master tenant2-net`

### **3.5 Add address to the vlan device:**

        sudo ip addr add 10.0.5.11/24 dev enp1s0.101

        # To ensure the kernel cleanly moves everything into the VRF's isolated routing table, it's always best practice to quickly bounce the link right after enslaving it:
        sudo ip link set dev enp1s0.101 down
        sudo ip link set dev enp1s0.101 up

### **3.6 Test the vrf route table:**

`ip route show vrf tenant2-net`

### **3.7 Create FRRConfiguration and RouteAdvertisements**

        # Note that 10.0.5.12 is the router IP, 65001 is the router ASN
         
        cat << 'EOF' > frr.yaml
        apiVersion: frrk8s.metallb.io/v1beta1
        kind: FRRConfiguration
        metadata:
          name: tenant2
          namespace: openshift-frr-k8s
          labels:
            routeAdvertisements: tenanats-frr
        spec:
          bgp:
            routers:
            - asn: 65002
              neighbors:
              - address: 10.0.5.12
                asn: 65001
                disableMP: true
                dualStackAddressFamily: false
                toReceive:
                  allowed:
                    mode: all
              vrf: tenant2-net
          nodeSelector: {}
        EOF

        oc apply -f frr.yaml

        cat << 'EOF' > routead.yaml
        apiVersion: k8s.ovn.org/v1
        kind: RouteAdvertisements
        metadata:
          name: advertise-vrf-lite
        spec:
          targetVRF: auto
          advertisements:
          - "PodNetwork"
          nodeSelector: {}
          frrConfigurationSelector:
            matchLabels:
              routeAdvertisements: tenanats-frr
          networkSelectors:
          - networkSelectionType: ClusterUserDefinedNetworks
            clusterUserDefinedNetworkSelector:
              networkSelector:
                matchLabels:
                  export: "true"
        EOF
        oc apply -f routead.yaml

## **Step 4: Router VM Provisioning & Configuration**

### **4.1 Create the OVS Bridge and Libvirt Network**

(It will be used by the router and the 2nd cluster)

        sudo ovs-vsctl add-br testpr

        cat << 'EOF' > testpr.xml
        <network>
          <name>testpr</name>
          <forward mode='bridge'/>
          <bridge name='testpr'/>
          <virtualport type='openvswitch'/>
        </network>
        EOF

        virsh net-define testpr.xml
        virsh net-start testpr
        virsh net-autostart testpr

### **4.2 Provision the Fedora Router VM**

        wget https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/images/Fedora-Server-Guest-Generic-43-1.6.x86_64.qcow2
        sudo cp Fedora-Server-Guest-Generic-43-1.6.x86_64.qcow2 /var/lib/libvirt/images/router-vm.qcow2

        virt-install \
          --name router-vm \
          --ram 2048 \
          --vcpus 2 \
          --os-variant fedora39 \
          --disk path=/var/lib/libvirt/images/router-vm.qcow2,format=qcow2,size=20 \
          --import \
          --network network=default \
          --network network=devpr \
          --network network=testpr \
          --graphics vnc \
          --noautoconsole

### **4.3 Enable SSH Access on the Router VM**

Connect to the VM using the VNC console (or `virsh console router-vm`) and
configure the initial user/root password if prompted.

Log in and edit the SSH configuration to allow remote access:
`sudo vi /etc/ssh/sshd_config`

Look for these two lines and ensure they look exactly like this (uncommented):

Plaintext

    PasswordAuthentication yes
    PermitRootLogin yes

Restart the SSH service to apply the changes: `sudo systemctl restart sshd`

Find the VM's assigned IP address on the default network and SSH into it for the
remaining configuration steps:

        virsh net-dhcp-leases default
        ssh root@<router-vm-ip>

### **4.4 Configure Hypervisor VLAN Trunking**

Edit the VM's XML (`virsh edit router-vm`) and add the VLAN trunk to the OVS
interface:

          <vlan trunk="yes">
            <tag id="101"/>
            <tag id="102"/>
            <tag id="103"/>
          </vlan>

Restart the VM and verify with `ovs-vsctl show`.

### **4.5 Configure Router OS Network (VRF/VLAN)** Inside the Router VM:

Inside the Router VM:

        sudo ip link add vrf-101 type vrf table 101
        sudo ip link set vrf-101 up

        sudo ip link add link enp2s0 name enp2s0.101 type vlan id 101
        sudo ip link set enp2s0.101 master vrf-101
        sudo ip addr add 10.0.5.12/24 dev enp2s0.101

        sudo ip link set enp2s0.101 up

### **4.6 Deploy FRR on the Router**

Inside the Router VM:

        sudo mkdir -p /var/lib/frr/etc/
        cat << 'EOF' | sudo tee /var/lib/frr/etc/frr.conf
        !
        frr version 9.1
        frr defaults traditional
        hostname UDMPRO
        log syslog informational
        service integrated-vtysh-config
        !
        vrf vrf-101
        exit-vrf
        !
        interface enp2s0.101
         vrf vrf-101
        exit
        !
        ip prefix-list ocp-hub seq 5 permit 10.0.1.0/24 le 32
        !
        route-map allow-ocp-hub permit 10
         match ip address prefix-list ocp-hub
        exit
        !
        router bgp 65001
        exit
        !
        router bgp 65001 vrf vrf-101
         bgp log-neighbor-changes
         no bgp default ipv4-unicast
         neighbor ocp-hub peer-group
         neighbor ocp-hub remote-as 65002
         neighbor 10.0.5.11 peer-group ocp-hub
         !
         address-family ipv4 unicast
          neighbor ocp-hub activate
          neighbor ocp-hub soft-reconfiguration inbound
          neighbor ocp-hub route-map allow-ocp-hub in
         exit-address-family
        exit
        !
        EOF

        echo "bgpd=yes" | sudo tee /var/lib/frr/etc/daemons

        sudo podman run -d --name frr --net=host --privileged \
          -v /var/lib/frr/etc:/etc/frr:Z \
          quay.io/frrouting/frr:9.1.0

## **Step 5: Testing & Debugging**

### **5.1 Verify Router BGP Session**

        sudo podman exec -it frr vtysh -c "show vrf"
        # Check the BGP communication
        sudo podman exec -it frr vtysh -c "show bgp vrf vrf-101 summary"
        # See the routing tables
        sudo podman exec -it frr vtysh -c "show bgp vrf vrf-101 ipv4 unicast"

### **5.2 Network Connectivity Test** From the OpenShift Worker Node:

`sudo ping -I enp1s0.101 10.0.5.12`

### **5.3 Pod Troubleshooting (Netshoot)**

Deploy this inside your tenant namespace to test routing from inside a pod:

        apiVersion: v1
        kind: Pod
        metadata:
          name: test-pod
          namespace: tenant1
        spec:
          containers:
          - name: main
            image: registry.redhat.io/rhel9/support-tools:latest
            command: ["sleep", "infinity"]

### Create the 2nd cluster

(I used [assisted-installer
SaaS](https://console.redhat.com/openshift/assisted-installer/clusters/~new) to
create SNO)\
Attached the VM to the libvirt default network and testpr (created earlier)

Create the tenant NS Create the CUDN

Check the VRF was created Create the vlan device Enslave the vlan to the VRF

        ip a
        sudo ip link add link enp7s0 name enp7s0.101 type vlan id 101
        ip vrf show
        sudo ip link set enp7s0.101 master tenant2-net
        sudo ip addr add 10.0.5.14/24 dev enp7s0.101
        sudo ip link set enp7s0.101 up

Complete the router configuration

        ssh root@<router-vm-ip>
        ip a
        sudo ip link add link enp3s0 name enp3s0.101 type vlan id 101
        sudo ip link set enp3s0.101 master vrf-101
        sudo ip addr add 10.0.6.12/24 dev enp3s0.101
        sudo ip link set enp3s0.101 up
        ip link show
        ip vrf show
        ip route show vrf  vrf-101

Update the `frr.conf`:

    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
    labels:
        machineconfiguration.openshift.io/role: worker
    name: 99-worker-bgp-vrf-sysctl
    spec:
    config:
        ignition:
          version: 3.2.0
        storage:
          files:
          - contents:
              source: data:,net.ipv4.tcp_l3mdev_accept%3D1%0A
            mode: 420
            path: /etc/sysctl.d/99-bgp-vrf.conf
