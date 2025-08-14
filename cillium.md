# Deploying Cilium with eBGP Peering to Cisco ACI

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Cilium Overview](#cilium-overview)
4. [Infrastructure Prerequisites](#infrastructure-prerequisites)
    - [UCS B-Series Configuration](#ucs-b-series-configuration)
    - [ACI Fabric Integration Architecture](#aci-fabric-integration-architecture)
5. [Kubernetes and OpenShift Cluster Preparation](#kubernetes-and-openshift-cluster-preparation)
    - [Node Provisioning and OS Configuration](#node-provisioning-and-os-configuration)
    - [Network Infrastructure Setup](#network-infrastructure-setup)
    - [High Availability and Node Network Configuration](#high-availability-and-node-network-configuration)
6. [Installing Cilium on Kubernetes](#installing-cilium-on-kubernetes)
7. [ACI L3Out Configuration](#aci-l3out-configuration)
    - [Creating Core ACI Objects](#creating-core-aci-objects)
    - [Bridge Domain Configuration](#bridge-domain-configuration)
    - [External EPG and Route Control](#external-epg-and-route-control)
8. [Establishing eBGP Peering](#establishing-ebgp-peering)
    - [Cilium BGP Configuration](#cilium-bgp-configuration)
    - [Per-Node BGP Session Setup](#per-node-bgp-session-setup)
    - [BGP Session Verification](#bgp-session-verification)
9. [BGP Route Advertisements](#bgp-route-advertisements)
    - [Pod CIDR Advertisement](#pod-cidr-advertisement)
    - [Service CIDR Advertisement](#service-cidr-advertisement)
    - [Route Monitoring and Troubleshooting](#route-monitoring-and-troubleshooting)
8. [Firewalling with Cilium](#firewalling-with-cilium)
9. [Conclusion](#conclusion)
10. [References](#references)

---

## Introduction
This document provides a comprehensive guide to deploying Cilium as a CNI on Kubernetes and OpenShift Container Platform (OCP), establishing eBGP peering with Cisco ACI L3Outs, advertising pod and service CIDRs, and integrating with UCS B-Series. It covers advanced topics such as scale optimization, firewalling, and high-availability node networking.

The guide addresses both vanilla Kubernetes and OpenShift deployments, highlighting the differences in configuration, security contexts, and operational procedures. OpenShift's enterprise-focused features such as Security Context Constraints (SCCs), operators, and integrated monitoring are covered alongside traditional Kubernetes approaches.

## Cilium Overview
Cilium is an advanced CNI (Container Network Interface) for Kubernetes, providing:

### Core Technologies
- **eBPF-based networking and security**: Leverages Linux kernel eBPF for high-performance packet processing
- **Native support for BGP**: Full BGP4 implementation with eBGP/iBGP peering capabilities
- **Fine-grained firewalling**: L3-L7 policy enforcement with DNS-aware rules
- **High scalability and performance**: 100Gbps+ throughput with sub-microsecond latency on modern infrastructure
- **Service mesh capabilities**: Native support for Envoy proxy integration
- **Observability**: Deep network and security observability via Hubble

### Architecture Components
- **cilium-agent**: Main daemon running on each node, handles eBPF program management
- **cilium-operator**: Cluster-wide operator managing CRDs and cluster-scope resources  
- **hubble-relay**: Aggregates flow data from all nodes for centralized observability
- **cilium-proxy**: Optional Envoy-based proxy for L7 policy enforcement

### Key Features for ACI Integration
- **BGP Control Plane**: Native BGP speaker supporting route redistribution and policy
- **IPAM Integration**: Flexible IP address management supporting cluster and external pools
- **Network Policy**: Kubernetes NetworkPolicy with Cilium-specific extensions
- **Load Balancing**: DSR (Direct Server Return) and NAT-based load balancing modes
- **Encryption**: Transparent encryption using IPSec or WireGuard protocols

## Infrastructure Prerequisites

Before deploying Kubernetes/OpenShift clusters with Cilium, the underlying infrastructure must be properly configured. This section covers the essential infrastructure components that must be deployed and configured prior to cluster installation.

## ACI Fabric 

### Physical Infrastructure Design
#### Topology Requirements:
```
Recommended Topology:
  Spine Switches: N9K-C9364C 
  Leaf Switches: N9K-C93180YC-EX or newer
  
Connection Standards:
  Spine-Leaf: 100G/400G QSFP+
  Leaf-UCS FI: 100G/400G QSFP+
```

- **UCS FI to ACI Leaf**: Use vPC pairs for redundancy
- **Minimum 2x 100G connections per FI to each leaf**
- **Cable both FIs to both leafs in vPC configuration**

## UCS B-Series Hardware

### Physical Connectivity and Hardware Specifications
- **Supported Platforms**: UCS B200 M5/M6, B480 M5, or newer
- **Fabric Interconnects**: 
  - **UCS-FI-64108**: 100G/400G QSFP28/QSFP-DD, 48 ports unified fabric (recommended)
  - **UCS-FI-64112**: 100G/400G/800G QSFP-DD, 32 ports for highest density/performance
  - **Legacy Support**: 6454/6536 still supported but not recommended for new deployments
- **Network Adapters**: VIC 1455/1457 for optimal performance and SR-IOV support
- **Memory**: DDR4-2933 or faster, minimum 128GB for production Kubernetes nodes
- **Storage**: NVMe SSDs for etcd and container runtime, minimum 1TB per node


### Intersight Configuration

#### 2. **Server Boot and Bios Policies**
```yaml
  Boot Policy: k8s-uefi-boot
    Boot Mode: UEFI
    Secure Boot: Enabled
    Boot Order:
      Local Disk (NVMe)
  
  BIOS Policy: k8s-performance-bios
    CPU Settings:
      Intel Turbo Boost: Enabled
      Intel Hyper-Threading: Enabled
      CPU Performance: Enterprise
      Energy Performance: Performance
    Memory Settings:
      NUMA Optimizations: Enabled
      Memory RAS: Maximum Performance
    PCIe Settings:
      SR-IOV: Enabled
      ARI Support: Enabled
```

#### 3. **vNIC Configuration**
```yaml
vNIC Templates:
    
  k8s-node-primary:
    Fabric: A  
    Redundancy Type: Primary
    VLAN: 200 (Node Network - BGP Peering to L3Out)
    MTU: 9000
    MAC Pool: k8s-node-mac-pool
    Pin Group: k8s-node-pin-group
    QoS Policy: k8s-high-priority
    
  k8s-node-secondary:
    Fabric: B
    Redundancy Type: Secondary
    VLAN: 200 (Node Network - BGP Peering to L3Out)
    MTU: 9000
    MAC Pool: k8s-node-mac-pool
    Pin Group: k8s-node-pin-group
    QoS Policy: k8s-high-priority
```

#### 4. **QoS and Network Control**
```yaml
QoS Policy: k8s-high-priority
  Priority: Platinum
  Burst Size: 65536
  Rate Limit: 0 (Unlimited)
  Host Control: Full
  
Network Control Policy: k8s-network-control
  CDP: Enabled
  LLDP Transmit: Enabled
  LLDP Receive: Enabled
  MAC Registration Mode: All Host VLANs
  Action on Uplink Fail: Link Down
  Forge MAC: Allow
```

## Kubernetes and OpenShift Cluster Preparation

### 1. **Node Provisioning and OS Configuration**

#### For Kubernetes Deployments:
   - Deploy Kubernetes nodes (bare metal) on UCS B-Series platforms
   - Disable swap and configure memory overcommit handling:

     ```bash
     swapoff -a
     echo 'vm.swappiness=1' >> /etc/sysctl.conf
     echo 'vm.overcommit_memory=1' >> /etc/sysctl.conf
     ```

#### For OpenShift Deployments:
   - Deploy OpenShift nodes using Red Hat CoreOS (RHCOS) on UCS B-Series platforms
   - Use OpenShift Installer (IPI) or User-Provisioned Infrastructure (UPI) methods
   - Configure ignition files for initial node setup and networking
   - Ensure proper entitlements and Red Hat subscriptions
   - Use Machine Config Operator for system-level configurations:

     ```yaml
     apiVersion: machineconfiguration.openshift.io/v1
     kind: MachineConfig
     metadata:
       labels:
         machineconfiguration.openshift.io/role: worker
       name: 99-worker-kernel-args
     spec:
       kernelArguments:
         - 'systemd.unified_cgroup_hierarchy=0'
         - 'cgroup_no_v1=all'
       config:
         ignition:
           version: 3.2.0
         systemd:
           units:
           - name: disable-swap.service
             enabled: true
             contents: |
               [Unit]
               Description=Disable swap
               [Service]
               Type=oneshot
               ExecStart=/usr/sbin/swapoff -a
               [Install]
               WantedBy=multi-user.target
     ```

### 2. **Network Infrastructure Setup**
   - Assign static IP addresses to nodes from dedicated IPpool (node network)
   - Ensure L3Out connectivity from node subnet to external networks via ACI fabric
   - Plan IP addressing scheme:

     - Node subnet: 10.1.0.0/24 (bridge-domain, single EPG or EPG per host with K8s contract)

     - Pod CIDR: 10.244.0.0/16 (Kubernetes default, L3out for pod traffic)
     - Pod CIDR: 10.128.0.0/14 (OpenShift default, L3out for pod traffic)

     - Service CIDR: 10.96.0.0/12 (Kubernetes default, L3out for service traffic)
     - Service CIDR: 172.30.0.0/16 (OpenShift default, L3out for service traffic)

   - Configure DNS resolution for cluster.local domain **Important**

### 3. **High Availability and Node Network Configuration**

Configure nodes with dual vNICs for network redundancy and implement active/backup bonding at the OS level for maximum availability.

#### UCS vNIC Configuration Prerequisites:
Before configuring OS-level bonding, ensure proper UCS vNIC setup:

```yaml
# UCS vNIC Templates (configured in Intersight)
k8s-node-primary:
  Fabric: A  
  Redundancy Type: Primary
  VLAN: 200 (Node Network - BGP Peering)
  MTU: 9000
  MAC Pool: k8s-node-mac-pool
  
k8s-node-secondary:
  Fabric: B
  Redundancy Type: Secondary  
  VLAN: 200 (Node Network - BGP Peering)
  MTU: 9000
  MAC Pool: k8s-node-mac-pool
```

#### Active/Backup Bonding Configuration (mode=1):

**For Kubernetes (systemd-networkd - Recommended):**
```bash
# /etc/systemd/network/10-bond0.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=active-backup
PrimaryReselectPolicy=always
MIIMonitorSec=100
UpDelaySec=200
DownDelaySec=200

# /etc/systemd/network/20-bond0-ens3.network (Primary interface)
[Match]
Name=ens3

[Network]
Bond=bond0
PrimarySlave=true

# /etc/systemd/network/21-bond0-ens4.network (Secondary interface)
[Match]
Name=ens4

[Network]
Bond=bond0

# /etc/systemd/network/30-bond0.network
[Match]
Name=bond0

[Network]
DHCP=no
Address=10.1.0.10/24  # Replace with actual node IP
Gateway=10.1.0.1
DNS=10.1.0.53
DNS=8.8.8.8

[Route]
Destination=10.244.0.0/16  # Kubernetes pod CIDR
Gateway=10.1.0.1
Metric=100

# Enable and restart networking
systemctl enable systemd-networkd
systemctl restart systemd-networkd
```

**For Kubernetes (Legacy ifcfg - RHEL/CentOS):**
```bash
# /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=static
IPADDR=10.1.0.10
NETMASK=255.255.255.0
GATEWAY=10.1.0.1
DNS1=10.1.0.53
DNS2=8.8.8.8
BONDING_OPTS="mode=active-backup miimon=100 primary=ens3"
ONBOOT=yes

# /etc/sysconfig/network-scripts/ifcfg-ens3
DEVICE=ens3
TYPE=Ethernet
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
ONBOOT=yes

# /etc/sysconfig/network-scripts/ifcfg-ens4
DEVICE=ens4
TYPE=Ethernet  
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
ONBOOT=yes
```

**For OpenShift (Machine Config Operator):**
```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-bond-config
spec:
  config:
    ignition:
      version: 3.2.0
    networkd:
      units:
      - name: 10-bond0.netdev
        contents: |
          [NetDev]
          Name=bond0
          Kind=bond
          
          [Bond]
          Mode=active-backup
          PrimaryReselectPolicy=always
          MIIMonitorSec=100
          UpDelaySec=200
          DownDelaySec=200
      - name: 20-bond0-ens3.network
        contents: |
          [Match]
          Name=ens3
          
          [Network]
          Bond=bond0
          PrimarySlave=true
      - name: 21-bond0-ens4.network
        contents: |
          [Match]
          Name=ens4
          
          [Network]
          Bond=bond0
      - name: 30-bond0.network
        contents: |
          [Match]
          Name=bond0
          
          [Network]
          DHCP=no
          Address=10.1.0.10/24  # Replace with actual node IP
          Gateway=10.1.0.1
          DNS=10.1.0.53
          DNS=8.8.8.8
          
          [Route]
          Destination=10.128.0.0/14  # OpenShift pod CIDR
          Gateway=10.1.0.1
          Metric=100
```

#### Network Performance Optimization:
```bash
# /etc/sysctl.d/99-kubernetes-network.conf
# Core networking
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 5000
net.core.netdev_budget = 600

# TCP optimization
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 90
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_slow_start_after_idle = 0

# Buffer sizes
net.core.rmem_default = 262144
net.core.rmem_max = 134217728
net.core.wmem_default = 262144
net.core.wmem_max = 134217728

# IPv4 routing optimization
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
```

#### Bond Status Verification:
```bash
# Check bond status
cat /proc/net/bonding/bond0

# Verify active slave
cat /sys/class/net/bond0/bonding/active_slave

# Monitor bond for failover testing
watch -n 1 'cat /sys/class/net/bond0/bonding/active_slave'

# Test failover by bringing down the primary interface
ip link set ens3 down
# Monitor for failover to secondary interface
# Bring primary back up
ip link set ens3 up
```

#### SR-IOV Configuration (Optional for High Performance):
```bash
# Enable SR-IOV in kernel
# /etc/default/grub additions
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt pci=realloc"

# Update grub and reboot
update-grub && reboot

# Enable VFs on physical interface
echo 8 > /sys/class/net/ens3/device/sriov_numvfs
echo 8 > /sys/class/net/ens4/device/sriov_numvfs
```

- Configure node anti-affinity for control plane components

## Installing Cilium on Kubernetes and OpenShift
### 1. **Prerequisites Verification**

#### For Kubernetes:
   ```bash
   # Verify kernel version and eBPF support
   uname -r
   ls /sys/fs/bpf/
   
   # Check for required kernel modules
   modprobe bpf
   lsmod | grep bpf
   
   # Verify Kubernetes cluster health
   kubectl get nodes -o wide
   kubectl get pods -n kube-system
   ```

#### For OpenShift:
   ```bash
   # Verify OpenShift cluster health
   oc get nodes -o wide
   oc get clusterversion
   oc get co  # Check cluster operators
   
   # Verify current CNI (typically OVN-Kubernetes)
   oc get network.operator cluster -o yaml
   
   # Check Security Context Constraints
   oc get scc
   ```

### 2. **Install Cilium CLI and Hubble**
   ```bash
   # Download Cilium CLI for Linux
   curl -L --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
   tar xzvf cilium-linux-amd64.tar.gz
   sudo mv cilium /usr/local/bin
   
   # Download Hubble CLI
   curl -L --remote-name https://github.com/cilium/hubble/releases/latest/download/hubble-linux-amd64.tar.gz
   tar xzvf hubble-linux-amd64.tar.gz
   sudo mv hubble /usr/local/bin
   
   # Verify installation
   cilium version --client
   hubble version --client
   ```

### 3. **Advanced Cilium Installation with BGP Support**

#### For Kubernetes:
   ```bash
   # Install Cilium with comprehensive BGP and observability features
   cilium install --version 1.15.0 \
     --set bgpControlPlane.enabled=true \
     --set bgpControlPlane.localASN=65001 \
     --set bgpControlPlane.announce.loadbalancerIP=true \
     --set bgpControlPlane.announce.podCIDR=true \
     --set bgpControlPlane.announce.nodeIP=true \
     --set hubble.relay.enabled=true \
     --set hubble.ui.enabled=true \
     --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}" \
     --set prometheus.enabled=true \
     --set operator.prometheus.enabled=true \
     --set kubeProxyReplacement=strict \
     --set k8sServiceHost=<API_SERVER_IP> \
     --set k8sServicePort=6443 \
     --set tunnel=disabled \
     --set ipam.mode=kubernetes \
     --set nativeRoutingCIDR=10.244.0.0/16 \
     --set autoDirectNodeRoutes=true \
     --set ipv4NativeRoutingCIDR=10.244.0.0/16 \
     --set enableIPv4Masquerade=false \
     --set enableIPv6Masquerade=false
   ```

#### For OpenShift:
   ```bash
   # Method 1: Using Cilium CLI with OpenShift-specific settings
   cilium install --version 1.15.0 \
     --set bgpControlPlane.enabled=true \
     --set bgpControlPlane.localASN=65001 \
     --set bgpControlPlane.announce.loadbalancerIP=true \
     --set bgpControlPlane.announce.podCIDR=true \
     --set bgpControlPlane.announce.nodeIP=true \
     --set hubble.relay.enabled=true \
     --set hubble.ui.enabled=true \
     --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}" \
     --set prometheus.enabled=false \
     --set operator.prometheus.enabled=false \
     --set kubeProxyReplacement=partial \
     --set k8sServiceHost=<API_SERVER_IP> \
     --set k8sServicePort=6443 \
     --set tunnel=disabled \
     --set ipam.mode=kubernetes \
     --set nativeRoutingCIDR=10.128.0.0/14 \
     --set autoDirectNodeRoutes=true \
     --set ipv4NativeRoutingCIDR=10.128.0.0/14 \
     --set enableIPv4Masquerade=false \
     --set enableIPv6Masquerade=false \
     --set securityContext.privileged=true \
     --set cni.chainingMode=none \
     --set cni.exclusive=false
   ```

#### Method 2: OpenShift Operator Installation:
   ```yaml
   # Create Cilium namespace and operator subscription
   apiVersion: v1
   kind: Namespace
   metadata:
     name: cilium
   ---
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: cilium
     namespace: cilium
   spec:
     targetNamespaces:
     - cilium
   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: cilium
     namespace: cilium
   spec:
     channel: stable
     name: cilium
     source: community-operators
     sourceNamespace: openshift-marketplace
   ```

#### OpenShift-Specific Security Context Configuration:
   ```yaml
   # Create custom SCC for Cilium
   apiVersion: security.openshift.io/v1
   kind: SecurityContextConstraints
   metadata:
     name: cilium
   allowHostDirVolumePlugin: true
   allowHostIPC: true
   allowHostNetwork: true
   allowHostPID: true
   allowHostPorts: true
   allowPrivilegedContainer: true
   allowedCapabilities:
   - '*'
   defaultAddCapabilities: null
   fsGroup:
     type: RunAsAny
   runAsUser:
     type: RunAsAny
   seLinuxContext:
     type: RunAsAny
   users:
   - system:serviceaccount:cilium:cilium-agent
   - system:serviceaccount:cilium:cilium-operator
   ```

### 4. **Configuration Parameters Explained**

#### Common Parameters:
   - `bgpControlPlane.enabled=true`: Enables native BGP speaker functionality
   - `tunnel=disabled`: Uses native routing instead of overlay (required for BGP)
   - `autoDirectNodeRoutes=true`: Automatically installs routes between nodes
   - `enableIPv4Masquerade=false`: Disables SNAT for direct routing
   - `nativeRoutingCIDR`: CIDR range for native routing (should match pod CIDR)

#### Kubernetes-Specific:
   - `kubeProxyReplacement=strict`: Replaces kube-proxy with eBPF implementation
   - `prometheus.enabled=true`: Enables Prometheus metrics collection
   - `nativeRoutingCIDR=10.244.0.0/16`: Default Kubernetes pod CIDR

#### OpenShift-Specific:
   - `kubeProxyReplacement=partial`: Partial kube-proxy replacement (safer for OpenShift)
   - `prometheus.enabled=false`: Use OpenShift's integrated monitoring instead
   - `nativeRoutingCIDR=10.128.0.0/14`: Default OpenShift pod CIDR
   - `securityContext.privileged=true`: Required for OpenShift SCC compliance
   - `cni.chainingMode=none`: Disable CNI chaining for direct replacement

### 5. **Post-Installation Verification**

#### For Kubernetes:
   ```bash
   # Check Cilium status
   cilium status --wait
   
   # Verify BGP is enabled
   kubectl get ciliumbgppeeringpolicies
   kubectl get ciliumbgpnodeconfigs
   
   # Check connectivity
   cilium connectivity test
   
   # Enable Hubble observability
   cilium hubble enable --ui
   ```

#### For OpenShift:
   ```bash
   # Check Cilium operator and pods
   oc get pods -n cilium
   oc get csv -n cilium  # Check Cluster Service Version
   
   # Verify Cilium configuration
   oc get cm cilium-config -n cilium -o yaml
   
   # Check BGP configuration
   oc get ciliumbgppeeringpolicies -A
   oc get ciliumbgpnodeconfigs -A
   
   # Verify OpenShift networking integration
   oc get network.operator cluster -o yaml
   oc get dns.operator cluster -o yaml
   
   # Test connectivity (may need SCC adjustments)
   cilium connectivity test --test-concurrency=1
   
   # Access Hubble UI through OpenShift route
   oc expose svc hubble-ui -n cilium
   oc get route hubble-ui -n cilium
   ```

#### OpenShift Network Operator Integration:
   ```yaml
   # Configure OpenShift to work with Cilium
   apiVersion: operator.openshift.io/v1
   kind: Network
   metadata:
     name: cluster
   spec:
     defaultNetwork:
       type: Cilium
       ciliumConfig:
         bgpControlPlane:
           enabled: true
           localASN: 65001
         hubble:
           enabled: true
         ipam:
           type: "Kubernetes"
   ```

## ACI L3Out Configuration

Before establishing BGP peering with Cilium, the ACI fabric must be properly configured with L3Outs, bridge domains, and external EPGs. This section covers the essential ACI infrastructure setup required for successful Cilium integration.

### 1. **Creating Core ACI Objects**

#### Step 1: Create Tenant and VRF
Configure the foundational ACI objects to host Kubernetes networks:

```yaml
# Tenant Configuration
Tenant: k8s-prod
Description: "Kubernetes Production Environment"

# VRF Configuration  
VRF: k8s-vrf
Policy Control: enforced
Direction: ingress
Route Target: auto-generated
```

#### Step 2: Create L3Out for External Connectivity
Configure the L3Out that will peer with Cilium nodes:

```yaml
# L3Out Configuration
Name: k8s-l3out
VRF: k8s-vrf
Domain: l3dom_k8s
BGP ASN: 65002

# Logical Node Profile
Logical Node Profile: k8s-border-leafs
Nodes: leaf-101, leaf-102
Router ID Loopback: enabled
BGP Route Reflector Client: disabled

# Logical Interface Profile
Logical Interface Profile: k8s-external-interfaces
Interface Type: SVI or Routed
MTU: 9000
```

### 2. **Bridge Domain Configuration**

Create bridge domains for each Kubernetes network segment:

#### Node Network Bridge Domain:
```yaml
Name: k8s-nodes-bd
VRF: k8s-vrf
Subnet: 10.1.0.0/24
Gateway: 10.1.0.1
Scope: Public, Shared
ARP Flooding: enabled
Unicast Routing: enabled
Unknown Unicast: proxy
```

#### Pod Network Bridge Domains:
```yaml
# Kubernetes Pods
Name: k8s-pods-bd
VRF: k8s-vrf
Subnet: 10.244.0.0/16
Gateway: 10.244.0.1
Scope: Public, Shared

# OpenShift Pods (if applicable)
Name: openshift-pods-bd
VRF: k8s-vrf
Subnet: 10.128.0.0/14
Gateway: 10.128.0.1
Scope: Public, Shared
```

#### Service Network Bridge Domains:
```yaml
# Kubernetes Services
Name: k8s-services-bd
VRF: k8s-vrf
Subnet: 10.96.0.0/12
Gateway: 10.96.0.1
Scope: Public, Shared

# OpenShift Services (if applicable)
Name: openshift-services-bd  
VRF: k8s-vrf
Subnet: 172.30.0.0/16
Gateway: 172.30.0.1
Scope: Public, Shared
```

### 3. **External EPG and Route Control**

Configure External EPG for route advertisement control:

```yaml
# External EPG Configuration
Name: k8s-external-epg
L3Out: k8s-l3out
Preferred Group: enabled

# Route Control Subnets (configured under External EPG)
Route Control Subnets:
  - 10.244.0.0/16 (K8s pods) - export-rtctrl, shared-rtctrl
  - 10.96.0.0/12 (K8s services) - export-rtctrl, shared-rtctrl
  - 10.128.0.0/14 (OpenShift pods) - export-rtctrl, shared-rtctrl  
  - 172.30.0.0/16 (OpenShift services) - export-rtctrl, shared-rtctrl
  - 0.0.0.0/0 (Default route) - import-security, shared-security
```

#### APIC Configuration Steps:
1. Navigate to **Tenants → k8s-prod → Networking → L3Outs**
2. Create **k8s-l3out** with proper VRF association
3. Configure **Logical Node Profile** with border leaf switches
4. Set up **BGP Peer Connectivity Profile** (will be configured per-node later)
5. Create **External EPG** with route control subnets
6. Associate bridge domains with appropriate EPGs

## Establishing eBGP Peering

With ACI L3Out infrastructure in place, the next step is to establish eBGP peering between Cilium nodes and ACI border leafs. This section covers both the Cilium configuration and the step-by-step process to bring up BGP sessions.

### 1. **Cilium BGP Configuration**

Deploy the Cilium BGP peering policy to enable eBGP sessions:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: aci-bgp-peering
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
  - localASN: 65001
    # Note: exportPodCIDR and advertisements will be configured in the next section
    neighbors:
    # Peer with border leaf-101
    - peerAddress: "192.168.100.1/32"  # Border leaf-101 SVI
      peerASN: 65002
      eBGPMultihop: 2
      holdTimeSeconds: 180
      keepaliveTimeSeconds: 60
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300
    # Peer with border leaf-102
    - peerAddress: "192.168.100.5/32"  # Border leaf-102 SVI  
      peerASN: 65002
      eBGPMultihop: 2
      holdTimeSeconds: 180
      keepaliveTimeSeconds: 60
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300
```

### 3. **Establishing eBGP Connections to Each Node**

### 2. **Per-Node BGP Session Setup**

#### Step 1: Verify Node Network Connectivity
Ensure basic IP connectivity before establishing BGP sessions:

```bash
# On each Kubernetes node, test connectivity to ACI border leaf SVIs
ping 192.168.100.1  # Border leaf-101 SVI
ping 192.168.100.5  # Border leaf-102 SVI

# Verify MTU and jumbo frames if configured
ping -M do -s 8972 192.168.100.1  # Test with 9000 MTU
```

#### Step 2: Configure ACI BGP Peers for Each Node
For each Kubernetes node, add BGP peer configuration in ACI:

**APIC Configuration Path:**
`Tenants → k8s-prod → Networking → L3Outs → k8s-l3out → Logical Node Profiles → k8s-border-leafs → BGP Peer Connectivity Profiles`

**Per-Node BGP Peer Configuration:**
```yaml
# Example for node k8s-worker-01 (IP: 10.1.0.10)
BGP_Peer_Configuration:
  Peer_Address: 10.1.0.10      # Kubernetes node IP
  Remote_ASN: 65001            # Cilium ASN
  Local_ASN: 65002             # ACI L3Out ASN  
  Connection_Mode: bidirectional
  Address_Family: ipv4-ucast
  Hold_Timer: 180              # Seconds
  Keepalive_Timer: 60          # Seconds
  BFD: enabled                 # Fast failure detection
  Send_Community: enabled      # For route tagging
  Send_Extended_Community: enabled
```

Repeat this configuration for each Kubernetes node in the cluster.

#### Step 3: Apply Cilium BGP Policy
Deploy the BGP peering policy:

```bash
# Kubernetes
kubectl apply -f cilium-bgp-peering-policy.yaml

# OpenShift
oc apply -f cilium-bgp-peering-policy.yaml
```

### 3. **BGP Session Verification**

#### Verify BGP Session Establishment
Monitor BGP session status from both sides:

```bash
# Kubernetes - Check BGP session status
kubectl exec -n kube-system ds/cilium -- cilium bgp peers
# Expected: "Established" state for both border leaf peers

# OpenShift - Check BGP session status  
oc exec -n cilium ds/cilium-agent -- cilium bgp peers
```

#### ACI Side Verification
```bash
# On ACI Border Leaf
show bgp ipv4 unicast summary vrf k8s-vrf

# Expected output example:
# Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
# 10.1.0.10       4 65001      45      42        8    0    0 00:15:23        0
# 10.1.0.11       4 65001      43      40        8    0    0 00:14:56        0

# Check specific neighbor details
show bgp ipv4 unicast neighbors 10.1.0.10 vrf k8s-vrf
```

#### Troubleshooting BGP Sessions
Common issues and solutions:

1. **Session Stuck in Active/Connect:**
   - Verify IP connectivity between nodes and border leafs
   - Check firewall rules allowing BGP port 179
   - Confirm ASN configuration matches on both sides

2. **Timer Mismatches:**
   - Ensure hold/keepalive timers match between Cilium and ACI
   - ACI: 180s hold, 60s keepalive
   - Cilium: Same timers in BGP policy

## BGP Route Advertisements

Once BGP sessions are established, configure and verify route advertisements for pod CIDRs and service networks. This section covers the configuration of BGP advertisements and monitoring.

### 1. **Pod CIDR Advertisement**

Update the Cilium BGP policy to enable pod CIDR advertisements:

```yaml
apiVersion: cilium.io/v2alpha1  
kind: CiliumBGPPeeringPolicy
metadata:
  name: aci-bgp-peering-with-pods
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
  - localASN: 65001
    exportPodCIDR: true
    exportPodCIDRMode: "local:true"  # Each node advertises only its local pod subnet
    neighbors:
    - peerAddress: "192.168.100.1/32"  # Border leaf-101 SVI
      peerASN: 65002
      eBGPMultihop: 2
      holdTimeSeconds: 180
      keepaliveTimeSeconds: 60
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300
    - peerAddress: "192.168.100.5/32"  # Border leaf-102 SVI
      peerASN: 65002
      eBGPMultihop: 2
      holdTimeSeconds: 180
      keepaliveTimeSeconds: 60
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300
    advertisements:
    - advertisementType: "PodCIDR"
      localPreference: 100
      communities:
        standard: ["65001:100"]
```

#### Pod CIDR Advertisement Modes:
- **exportPodCIDRMode: "local:true"**: Each node advertises only its allocated pod subnet (/24 from cluster /16)
  - Benefits: Reduces BGP table size, improves convergence, enables traffic engineering
- **exportPodCIDRMode: "false"** (default): All nodes advertise the entire cluster pod CIDR

#### Verify Pod CIDR Advertisement:
```bash
# Check advertised routes from Cilium
kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised

# Expected output: Each node advertises its local pod CIDR (e.g., 10.244.1.0/24)

# Verify routes received on ACI border leafs
show bgp ipv4 unicast vrf k8s-vrf | include 10.244
# Expected: Per-node pod CIDR routes
```

### 2. **Service CIDR Advertisement**

Configure service advertisements by updating the BGP policy:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: aci-bgp-peering-with-services
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
  - localASN: 65001
    exportPodCIDR: true
    exportPodCIDRMode: "local:true"
    neighbors:
    - peerAddress: "192.168.100.1/32"
      peerASN: 65002
      eBGPMultihop: 2
      holdTimeSeconds: 180
      keepaliveTimeSeconds: 60
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300
    - peerAddress: "192.168.100.5/32"
      peerASN: 65002
      eBGPMultihop: 2
      holdTimeSeconds: 180
      keepaliveTimeSeconds: 60
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300
    advertisements:
    - advertisementType: "PodCIDR"
      localPreference: 100
      communities:
        standard: ["65001:100"]
    - advertisementType: "Service"
      service:
        matchLabels:
          advertise-via-bgp: "true"  # Only advertise services with this label
      localPreference: 200
      communities:
        standard: ["65001:200"]
```

#### Service Advertisement Modes:
- **Local Traffic Policy**: Only nodes with pods advertise service IP as /32 (recommended for performance)
- **Cluster Traffic Policy**: All nodes advertise entire service CIDR

#### Example Service Configuration:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    advertise-via-bgp: "true"  # Required for BGP advertisement
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Direct routing to pod nodes
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

#### Verify Service Advertisement:
```bash
# Create test service
kubectl create service loadbalancer test-svc --tcp=80:80
kubectl label service test-svc advertise-via-bgp=true

# Check if service IP is advertised
kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised | grep -A5 -B5 test-svc

# Verify on ACI side
show bgp ipv4 unicast vrf k8s-vrf | include 10.96
```

### 3. **Route Monitoring and Troubleshooting**

#### Comprehensive Route Verification:
```bash
# Kubernetes - Check all BGP routes
kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised
kubectl exec -n kube-system ds/cilium -- cilium bgp routes received

# OpenShift - Check all BGP routes
oc exec -n cilium ds/cilium-agent -- cilium bgp routes advertised
oc exec -n cilium ds/cilium-agent -- cilium bgp routes received

# ACI - Verify received routes
show bgp ipv4 unicast summary vrf k8s-vrf
show bgp ipv4 unicast vrf k8s-vrf
show ip route vrf k8s-vrf 10.244.0.0/16
show ip route vrf k8s-vrf 10.96.0.0/12
```

#### Production Monitoring Script:
```bash
#!/bin/bash
# BGP Route Advertisement Monitoring
while true; do
  echo "=== BGP Session Status $(date) ==="
  kubectl exec -n kube-system ds/cilium -- cilium bgp peers | grep -E "(Peer|Established|Active)"
  
  echo "=== Pod CIDR Routes ==="
  kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised | grep -c "10.244"
  
  echo "=== Service Routes ==="
  kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised | grep -c "10.96"
  
  sleep 30
done
```

#### Common Troubleshooting Issues:

1. **Routes Not Being Advertised:**
   - Verify `exportPodCIDR: true` in BGP policy
   - Check service labels for `advertise-via-bgp: "true"`
   - Confirm BGP sessions are in "Established" state

2. **Partial Route Advertisement:**
   - Check `exportPodCIDRMode` setting
   - Verify all nodes have BGP sessions established
   - Review ACI External EPG route control subnets

3. **Route Black Holes:**
   - Verify bridge domain associations in ACI
   - Check ACI route table entries match advertised routes
   - Confirm proper next-hop reachability
```bash
# Kubernetes
kubectl exec -n kube-system ds/cilium -- cilium bgp peers
kubectl exec -n kube-system ds/cilium -- cilium bgp routes

# OpenShift  
oc exec -n cilium ds/cilium-agent -- cilium bgp peers
oc exec -n cilium ds/cilium-agent -- cilium bgp routes

# ACI APIC/Border Leaf
show bgp ipv4 unicast summary vrf k8s-vrf


## Advanced ACI L3Out Best Practices and Optimization

### 1. **BGP Configuration Best Practices**

#### BGP AS Design:
- **ACI Fabric AS**: Use private ASN range (64512-65534) for ACI fabric
- **External AS**: Coordinate with network team for external device ASNs
- **BGP Route Reflectors**: Deploy minimum 2 RR spines per pod for redundancy

#### BGP Timer Optimization:
```yaml
# Optimized BGP configuration for production
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: aci-ebgp-production
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
  - localASN: 65001
    exportPodCIDR: true
    exportPodCIDRMode: "local:true"  # Per-node advertisements for scale
    neighbors:
    - peerAddress: <ACI_L3OUT_SVI_IP>/32
      peerASN: 65002
      eBGPMultihop: 2
      connectRetryTimeSeconds: 30      # Increased for stability
      holdTimeSeconds: 180             # Cisco recommended value
      keepaliveTimeSeconds: 60         # 1/3 of hold time
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 300        # Extended for large tables
      advertisements:
      - advertisementType: "PodCIDR"
        localPreference: 100
        communities:
          standard: ["65001:100"]
      - advertisementType: "Service"
        service:
          matchLabels:
            advertise-via-bgp: "true"  # Selective service advertisement
        localPreference: 200
        communities:
          standard: ["65001:200"]
```

### 2. **ACI L3Out Design Patterns**

#### External EPG Configuration:
Following Cisco best practices for route control and security:

```yaml
# ACI External EPG Subnet Scopes (configured in APIC)
External_EPG_Subnets:
  # For contract classification (longest prefix match)
  external_subnets_for_external_epg:
    - 10.244.0.0/16    # Kubernetes pods (import routes)
    - 10.96.0.0/12     # Kubernetes services (import routes)
    - 10.128.0.0/14    # OpenShift pods (import routes)
    - 172.30.0.0/16    # OpenShift services (import routes)
  
  # For route advertisement control (exact match)
  export_route_control_subnet:
    - 10.244.0.0/16    # Advertise pod networks
    - 10.96.0.0/12     # Advertise service networks
    - 10.128.0.0/14    # Advertise OpenShift pod networks
    - 172.30.0.0/16    # Advertise OpenShift service networks
```

### 3. **VRF Tag and Loop Prevention**

#### ACI VRF Tag Configuration:
```yaml
# ACI VRF configuration for loop prevention
VRF_Settings:
  vrf_tag: 4294967295  # Default ACI VRF tag
  policy_control_enforcement: "ingress"  # Default direction
  route_target_auto: true
```

#### Route Filtering Best Practices:
- Use **BGP communities** for granular route control
- Implement **route-maps** for prefix filtering
- Configure **prefix limits** to prevent route table overflow
- Enable **BFD** for fast convergence

### 4. **Scale Optimization Techniques**

#### Route Summarization:
```yaml
# Cilium route summarization for large deployments
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: scale-optimized-bgp
spec:
  bgpInstances:
  - name: "production-instance"
    localASN: 65001
    peers:
    - name: "aci-border-leafs"
      peerASN: 65002
      peerAddress: 10.1.1.0/30  # Border leaf SVI subnet
    advertisements:
    # Advertise summarized routes instead of individual /24s
    - advertisementType: "PodCIDR"
      selector:
        matchLabels:
          node-role: "worker"
      attributes:
        localPreference: 100
        communities:
          standard: ["65001:summary"]
```

#### BGP Table Size Management:
- **Local PodCIDR Mode**: Reduces BGP table from thousands of /24s to per-node advertisements
- **Route Filtering**: Use import/export policies to limit unnecessary routes
- **Community-based Policies**: Implement hierarchical route policies

### 5. **High Availability and Monitoring**

#### BGP Session Monitoring:
```bash
# Cilium BGP monitoring commands
# Check BGP session status
kubectl exec -n kube-system ds/cilium -- cilium bgp peers

# Monitor route advertisements with communities
kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised

# Verify route reception
kubectl exec -n kube-system ds/cilium -- cilium bgp routes received
```

### 7. **Troubleshooting and Validation**

#### Common Issues and Solutions:

**BGP Session Flapping:**
- Check MTU alignment between Cilium nodes and ACI
- Verify BGP timer settings match ACI configuration
- Validate network connectivity and firewall rules

**Route Advertisement Problems:**
- Confirm ACI External EPG subnet scopes
- Verify Cilium service annotation for BGP advertisement
- Check BGP community configuration alignment

**Performance Issues:**
- Enable BGP graceful restart for maintenance windows
- Implement route dampening for unstable routes
- Use BFD for fast failure detection

#### Diagnostic Commands:
```bash
# ACI border leaf diagnostics
leaf# show bgp ipv4 unicast summary vrf <vrf-name>
leaf# show ip route vrf <vrf-name> 10.244.0.0/16
leaf# show interface brief

# Cilium diagnostics
kubectl exec -n kube-system ds/cilium -- cilium status
kubectl exec -n kube-system ds/cilium -- cilium bgp routes
cilium connectivity test --test bgp-advertisement
```

This integration ensures enterprise-grade reliability and follows Cisco's proven L3Out design principles while leveraging Cilium's advanced eBPF networking capabilities.

#### Layer 3/4 Network Policies (Kubernetes):
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: secure-database-access
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: database
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: web-server
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
  - fromCIDR:
    - "10.1.0.0/24"  # Management network
    toPorts:
    - ports:
      - port: "22"
        protocol: TCP
  egress:
  - toServices:
    - k8sService:
        serviceName: auth-service
        namespace: auth
  - toCIDR:
    - "0.0.0.0/0"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

#### Layer 3/4 Network Policies (OpenShift):
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: secure-database-access-openshift
  namespace: production
  annotations:
    # OpenShift-specific annotations for project integration
    openshift.io/description: "Cilium policy for database security"
spec:
  endpointSelector:
    matchLabels:
      app: database
      deployment: production
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: web-server
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
  # Allow OpenShift service mesh sidecar communication
  - fromEndpoints:
    - matchLabels:
        app: istio-proxy
    toPorts:
    - ports:
      - port: "15090"  # Envoy admin port
        protocol: TCP
  egress:
  - toServices:
    - k8sService:
        serviceName: auth-service
        namespace: auth
  # Allow access to OpenShift internal services
  - toCIDR:
    - "172.30.0.0/16"  # OpenShift service network
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

#### Layer 7 HTTP Policy:
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-gateway-l7-policy
spec:
  endpointSelector:
    matchLabels:
      app: api-gateway
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"
        - method: "POST"
          path: "/api/v1/users"
          headers:
          - "Content-Type: application/json"
```

### 2. **DNS-Aware Security Policies**
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-based-egress
spec:
  endpointSelector:
    matchLabels:
      app: web-service
  egress:
  - toFQDNs:
    - matchName: "api.external-service.com"
    - matchPattern: "*.cdn.example.com"
  - toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

### 3. **Service Mesh Integration**
```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: rate-limiting
spec:
  services:
  - name: my-service
    namespace: default
  resources:
  - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
    name: envoy-prometheus-metrics-listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 9964
```

### 4. **Encryption and Traffic Protection**
#### IPSec Configuration:
```bash
# Enable transparent encryption
cilium install \
  --set encryption.enabled=true \
  --set encryption.type=ipsec \
  --set encryption.ipsec.keyFile=/etc/ipsec/keys
```

#### WireGuard Configuration:
```bash
# Alternative: WireGuard encryption (Linux 5.6+)
cilium install \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```

### 5. **Security Monitoring and Compliance**

#### Hubble Security Observability (Kubernetes):
```bash
# Enable comprehensive security monitoring
cilium hubble enable \
  --ui \
  --relay \
  --metrics "dns,drop,tcp,flow,icmp,http"

# Access Hubble UI
cilium hubble ui
```

#### Hubble Security Observability (OpenShift):
```bash
# Enable Hubble with OpenShift integration
cilium hubble enable \
  --ui \
  --relay \
  --metrics "dns,drop,tcp,flow,icmp,http"

# Create OpenShift route for Hubble UI
oc expose svc hubble-ui -n cilium
oc get route hubble-ui -n cilium

# Integrate with OpenShift monitoring
oc create -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hubble-metrics
  namespace: cilium
spec:
  endpoints:
  - port: hubble-metrics
    interval: 30s
  selector:
    matchLabels:
      k8s-app: hubble
EOF
```

#### Policy Audit Mode (Common):
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: audit-policy
  annotations:
    policy.cilium.io/audit-mode: "true"
    # OpenShift-specific: integrate with audit logging
    openshift.io/audit-policy: "true"
spec:
  endpointSelector:
    matchLabels:
      app: sensitive-app
  # Policy rules that will be audited but not enforced
```

#### OpenShift-Specific Security Integration:
```yaml
# Configure Cilium to work with OpenShift Security Context Constraints
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-openshift-config
  namespace: cilium
data:
  enable-openshift-scc: "true"
  openshift-skip-crd-creation: "false"
  # Integration with OpenShift Pod Security Standards
  pod-security-standards: "restricted"
```

### Summary:

For manual ACI configuration or infrastructure as code automation, create the following objects in APIC:

#### Essential ACI Objects:
1. **Tenant and VRF**: k8s-prod tenant with k8s-vrf (policy control enforced)
2. **L3Out**: k8s-l3out with BGP ASN 65002, associated with k8s-vrf
3. **Border Leaf Configuration**: BGP peers on leaf-101/102 with Router ID loopback
4. **External EPG**: k8s-external-epg with route control subnets for pod/service CIDRs
5. **Bridge Domains**: Create BDs for node, pod, and service networks with proper scoping

#### Key Configuration Parameters:
- **BGP Timers**: Hold=180s, Keepalive=60s (production recommended)
- **Route Control**: Export-only for pod/service CIDRs, import for management networks
- **VRF Policy**: Ingress policy control with enforced direction
- **External EPG Subnets**: Use export-rtctrl scope for Kubernetes networks

## Conclusion
Deploying Cilium with eBGP peering to Cisco ACI L3Outs represents a sophisticated approach to enterprise Kubernetes networking that bridges the gap between cloud-native workloads and traditional data center infrastructure. This comprehensive integration enables organizations to leverage the advanced networking capabilities of both platforms while maintaining the security, observability, and scalability requirements of modern enterprise environments.

**Enterprise-Grade Security**: Cilium's eBPF-based policy enforcement combined with ACI's hardware-accelerated contract system provides defense-in-depth security at multiple layers. The integration supports L3-L7 policy enforcement, DNS-aware rules, and transparent encryption, enabling comprehensive microsegmentation and compliance with enterprise security frameworks. OpenShift's Security Context Constraints (SCCs) provide additional security layers that integrate seamlessly with Cilium's policy model.

**Platform-Specific Optimizations**: 

- **Kubernetes**: Full kube-proxy replacement with strict eBPF implementation for maximum performance
- **OpenShift**: Partial kube-proxy replacement ensuring compatibility with OpenShift's integrated monitoring and service mesh capabilities

**High-Performance Infrastructure**: The UCS B-Series integration with SR-IOV, active-backup bonding, and optimized network stack tuning delivers the performance characteristics required for demanding workloads on both platforms. The proper configuration of fabric interconnects, ACI leaf switches, and vPC ensures sub-second failover times and high availability.

### Operational Excellence and Observability
The Hubble integration provides unprecedented visibility into cluster networking behavior on both Kubernetes and OpenShift platforms, enabling proactive monitoring and rapid troubleshooting. Combined with ACI's comprehensive fabric monitoring and OpenShift's integrated monitoring stack, operators gain end-to-end observability from container networking through the physical infrastructure. OpenShift's native integration with Prometheus and Grafana enhances the monitoring capabilities with platform-specific dashboards and alerts.

### Scalability and Future-Proofing
The architecture scales horizontally through BGP route optimization and vertically through performance tuning. The modular design enables incremental adoption and integration with additional enterprise systems such as service mesh, storage networks, and external connectivity providers.

## References and Additional Resources
### Official Documentation
- [Cilium Documentation](https://docs.cilium.io/en/stable/) - Comprehensive Cilium configuration and API reference
- [Cilium BGP Control Plane](https://docs.cilium.io/en/stable/network/bgp-control-plane/) - Detailed BGP implementation guide
- [Cisco ACI BGP L3Out Configuration Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/4-x/l3out/b_Cisco_APIC_Layer_3_Outside_Configuration_Guide.html) - ACI L3Out comprehensive configuration
- [UCS B-Series Installation and Service Guides](https://www.cisco.com/c/en/us/support/servers-unified-computing/ucs-b-series-blade-servers/products-installation-guides-list.html) - Hardware configuration reference

### Advanced Technical Resources
- [Kubernetes Networking Concepts](https://kubernetes.io/docs/concepts/cluster-administration/networking/) - CNI and cluster networking fundamentals
- [OpenShift Networking](https://docs.openshift.com/container-platform/4.12/networking/understanding-networking.html) - OpenShift-specific networking concepts
- [Hubble Observability Platform](https://docs.cilium.io/en/stable/gettingstarted/hubble/) - Network observability and monitoring
- [eBPF Programming Guide](https://ebpf.io/) - Understanding eBPF for network programming
- [Linux Advanced Routing & Traffic Control](https://lartc.org/) - Network stack optimization and tuning
- [OpenShift Machine Config Operator](https://docs.openshift.com/container-platform/4.12/post_installation_configuration/machine-configuration-tasks.html) - Node-level configuration management

### Cisco Technical Resources
- [Cisco Intersight Documentation](https://intersight.com/help/) - UCS management and automation
- [ACI Fabric Design Best Practices](https://www.cisco.com/c/en/us/solutions/collateral/data-center-virtualization/application-centric-infrastructure/white-paper-c11-737909.html) - Infrastructure design principles
- [UCS Performance Tuning Guide](https://www.cisco.com/c/en/us/support/docs/servers-unified-computing/ucs-manager/200501-UCS-Performance-Tuning-Best-Practices.html) - Hardware optimization

### Community and Open Source
- [Cilium GitHub Repository](https://github.com/cilium/cilium) - Source code and issue tracking
- [Cilium Slack Community](https://cilium.herokuapp.com/) - Community support and discussions
- [CNCF Networking SIG](https://github.com/kubernetes/community/tree/master/sig-network) - Kubernetes networking special interest group
- [OpenShift Commons](https://commons.openshift.org/) - OpenShift community resources and best practices
- [Red Hat Customer Portal](https://access.redhat.com/) - OpenShift enterprise support and documentation
