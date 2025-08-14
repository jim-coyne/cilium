# Deploying Cilium with eBGP Peering to Cisco ACI L3Outs: A Comprehensive Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Cilium Overview](#cilium-overview)
4. [Infrastructure Prerequisites](#infrastructure-prerequisites)
    - [UCS B-Series Configuration](#ucs-b-series-configuration)
    - [ACI Fabric Integration Architecture](#aci-fabric-integration-architecture)
    - [Kubernetes Node Network Configuration (Active/Backup)](#kubernetes-node-network-configuration-activebackup)
5. [Kubernetes and OpenShift Cluster Preparation](#kubernetes-and-openshift-cluster-preparation)
6. [Installing Cilium on Kubernetes](#installing-cilium-on-kubernetes)
7. [Configuring eBGP Peering with ACI L3Outs](#configuring-ebgp-peering-with-aci-l3outs)
    - [Advertising Pod and Service CIDRs](#advertising-pod-and-service-cidrs)
    - [Bridge Domain for Node/Compute Network](#bridge-domain-for-nodecompute-network)
    - [Scale Configurations and /32 Route Optimization](#scale-configurations-and-32-route-optimization)
8. [Firewalling with Cilium](#firewalling-with-cilium)
9. [Conclusion](#conclusion)
10. [References](#references)

---

## Introduction
This document provides a comprehensive guide to deploying Cilium as a CNI on Kubernetes and OpenShift Container Platform (OCP), establishing eBGP peering with Cisco ACI L3Outs, advertising pod and service CIDRs, and integrating with UCS B-Series. It covers advanced topics such as scale optimization, firewalling, and high-availability node networking.

The guide addresses both vanilla Kubernetes and OpenShift deployments, highlighting the differences in configuration, security contexts, and operational procedures. OpenShift's enterprise-focused features such as Security Context Constraints (SCCs), operators, and integrated monitoring are covered alongside traditional Kubernetes approaches.

### Platform-Specific Requirements
#### OpenShift Considerations:
- OpenShift 4.10+ with cluster-admin privileges
- Access to Red Hat OperatorHub for Cilium operator installation
- Understanding of Security Context Constraints (SCCs)
- Familiarity with OpenShift networking (OVN-Kubernetes default)
- Machine Config Operator (MCO) access for node-level configurations

#### Kubernetes Considerations:
- Vanilla Kubernetes 1.23+ or managed distributions (EKS, GKE, AKS)
- CNI replacement capabilities (removing existing CNI)

### Network Infrastructure
- Cisco ACI fabric, bridgedomain/EPG for node network, L3Outs for services and pods
- UCS B-Series servers (B200 M5/M6, B480 M5, or newer recommended)
- Fabric Interconnects (64108 or newer for optimal performance)
- ACI fabric with APIC version 4.2+ (5.x+ recommended)
- BGP ASN ranges planned (private ASNs - example: 64512-65534)

### Access and Permissions
- Administrative access to all systems (root/sudo on K8s nodes)
- APIC admin access or tenant admin with L3Out privileges
- Intersight account admin access for UCS management
- Docker registry access for Cilium container images

### Technical Knowledge
- Deep familiarity with BGP routing protocols and AS path manipulation
- Understanding of Kubernetes/OpenShift networking, CNI, and kube-proxy/OVN-Kubernetes
- OpenShift-specific: Security Context Constraints, Operators, and Machine Configs
- ACI fabric concepts: tenants, VRFs, bridge domains, EPGs, contracts
- Linux networking: iptables, netfilter, eBPF, and network namespaces
- Container networking fundamentals and overlay vs underlay concepts
- Red Hat CoreOS administration and systemd service management (for OpenShift)

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

## ACI Fabric Integration Architecture

### Physical Infrastructure Design
#### Topology Requirements:
```
Recommended Topology:
  Spine Switches: N9K-C9364C 
  Leaf Switches: N9K-C93180YC-EX or newer
  APIC Controllers: 3-node cluster 
  
Connection Standards:
  Spine-Leaf: 100G/400G QSFP+
  Leaf-UCS FI: 100G/400G QSFP+
```

#### Cable Planning and Physical Connections:
- **UCS FI to ACI Leaf**: Use vPC pairs for redundancy
- **Minimum 2x 100G connections per FI to each leaf**
- **Cable both FIs to both leafs in vPC configuration**

## UCS B-Series Configuration

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

### 3. **High Availability and Redundancy**
   - Configure nodes with dual vNICs for network redundancy
   - Implement active/backup bonding at OS level (mode=1):
   
   **Active/Backup Bonding Configuration (mode=1):**
   
   #### For Kubernetes (systemd-networkd - Recommended):
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
   Address=10.1.0.10/24  # Replace X with actual node IP
   Gateway=10.1.0.1
   DNS=10.1.0.53
   DNS=8.8.8.8
   
   # Enable and restart networking
   systemctl enable systemd-networkd
   systemctl restart systemd-networkd
   ```
   
   #### For OpenShift (Machine Config Operator):
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
   ```
   
   **Bond Status Verification:**
   ```bash
   # Check bond status
   cat /proc/net/bonding/bond0
   
   # Verify active slave
   cat /sys/class/net/bond0/bonding/active_slave
   
   # Monitor bond for failover testing
   watch -n 1 'cat /sys/class/net/bond0/bonding/active_slave'
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

## Configuring eBGP Peering with ACI L3Outs
### 1. **ACI L3Out Detailed Configuration**
#### APIC Configuration Steps:
1. **Create Tenant and VRF:**
   ```
   Tenant: k8s-prod
   VRF: k8s-vrf
   Route Target: auto-generated
   ```

2. **Create L3Out:**
   ```
   Name: k8s-l3out
   VRF: k8s-vrf
   Domain: l3dom_k8s
   BGP ASN: 65002
   ```

3. **Configure BGP Peer Connectivity Profile:**
   ```
   Logical Node Profile: k8s-nodes
   Logical Interface Profile: k8s-interfaces
   BGP Peer Connectivity Profile:
     - Peer IP: <each_k8s_node_ip>
     - Remote ASN: 65001
     - Local ASN: 65002
     - Hold Interval: 90
     - Keepalive Interval: 30
     - Connection Mode: bidirectional
   ```

4. **Create External EPG:**
   ```
   Name: k8s-external-epg
   Preferred Group: enabled
   Subnets:
     - 10.244.0.0/16 (Kubernetes pod CIDR) - Import/Export Route Control
     - 10.96.0.0/12 (Kubernetes service CIDR) - Import/Export Route Control
     - 10.128.0.0/14 (OpenShift pod CIDR) - Import/Export Route Control  
     - 172.30.0.0/16 (OpenShift service CIDR) - Import/Export Route Control
   ```

### 2. **Advanced Cilium BGP Configuration**
Create a comprehensive `CiliumBGPPeeringPolicy`:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: aci-ebgp-peering-advanced
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
  - localASN: 65001
    exportPodCIDR: true
    exportPodCIDRMode: "local:true"
    serviceSelector:
      matchExpressions:
      - key: somekey
        operator: NotIn
        values: ['never-used-value']
    neighbors:
    - peerAddress: <ACI_L3OUT_SVI_IP>/32
      peerASN: 65002
      eBGPMultihop: 2
      connectRetryTimeSeconds: 10
      holdTimeSeconds: 90
      keepaliveTimeSeconds: 30
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 120
      advertisements:
      - advertisementType: "PodCIDR"
        localPreference: 100
        communities:
          standard: ["65001:100"]
      - advertisementType: "Service"
        service:
          matchExpressions:
          - key: type
            operator: In
            values: ['LoadBalancer', 'NodePort']
        localPreference: 200
        communities:
          standard: ["65001:200"]
```

### 3. **BGP Route Policies and Communities**
Configure advanced BGP policies using `CiliumBGPAdvertisement`:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPAdvertisement
metadata:
  name: pod-cidr-advertisement
spec:
  advertisements:
  - advertisementType: "PodCIDR"
    attributes:
      localPreference: 100
      communities:
        standard: ["65001:100"]
      aspath:
        prepends: 1
  - advertisementType: "Service"
    selector:
      matchLabels:
        advertise: "bgp"
    attributes:
      localPreference: 200
      communities:
        standard: ["65001:200", "65001:300"]
```

### 4. **Monitoring and Troubleshooting BGP Sessions**

#### For Kubernetes:
```bash
# Check BGP session status
kubectl get ciliumbgpnodeconfigs -o yaml

# View BGP routes
kubectl exec -n kube-system ds/cilium -- cilium bgp routes

# Debug BGP sessions
kubectl exec -n kube-system ds/cilium -- cilium bgp peers

# Monitor route advertisements
kubectl logs -n kube-system ds/cilium | grep bgp
```

#### For OpenShift:
```bash
# Check BGP session status (Cilium deployed in cilium namespace)
oc get ciliumbgpnodeconfigs -o yaml -n cilium

# View BGP routes from Cilium pods
oc exec -n cilium ds/cilium-agent -- cilium bgp routes

# Debug BGP sessions
oc exec -n cilium ds/cilium-agent -- cilium bgp peers

# Monitor route advertisements using OpenShift logging
oc logs -n cilium ds/cilium-agent | grep bgp

# Check Cilium configuration via ConfigMap
oc get cm cilium-config -n cilium -o yaml

# Verify node readiness and BGP peer status
oc describe nodes | grep -A5 -B5 "Network Unavailable"
```

#### OpenShift Monitoring Integration:
```yaml
# Create ServiceMonitor for Cilium metrics in OpenShift
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-agent
  namespace: cilium
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    port: prometheus
    scheme: http
  selector:
    matchLabels:
      app.kubernetes.io/name: cilium-agent
```

### 3. **Advanced Pod and Service CIDR Advertisement**
#### Pod CIDR Advertisement Configuration:
Cilium advertises pod networks using two different modes:

**exportPodCIDRMode Settings:**
- `exportPodCIDRMode: "local:true"`: Each node advertises only its own allocated pod subnet (/24 from /16 cluster CIDR)
  - Example: Node1 advertises 10.244.1.0/24, Node2 advertises 10.244.2.0/24
  - Benefits: Reduces BGP table size, improves convergence time, enables per-node traffic engineering
- `exportPodCIDRMode: "false"` (default): All nodes advertise the entire cluster pod CIDR
  - Example: All nodes advertise 10.244.0.0/16

#### Service Advertisement Configuration:
Service advertisement behavior depends on `externalTrafficPolicy` and load balancer configuration:

**Local Traffic Policy (Recommended for Performance):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    service.cilium.io/lb-ipam-ips: "10.96.100.1"  # Optional: specific IP
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Only nodes with pods advertise service IP as /32
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
---
# BGP advertisement behavior:
# - Nodes WITH web pods: Advertise 10.96.100.1/32
# - Nodes WITHOUT web pods: Do not advertise this service IP
# - Result: Traffic goes directly to nodes with endpoints, no kube-proxy forwarding
```

**Cluster Traffic Policy (Default):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service-cluster
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster  # All nodes advertise entire service subnet
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
---
# BGP advertisement behavior:
# - All nodes: Advertise entire service CIDR (e.g., 10.96.0.0/12)
# - Result: Traffic can land on any node, kube-proxy forwards to endpoints
```

**Advanced BGP Service Advertisement Control:**
```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPAdvertisement
metadata:
  name: selective-service-advertisement
spec:
  advertisements:
  - advertisementType: "Service"
    service:
      matchLabels:
        advertise-via-bgp: "true"  # Only advertise services with this label
    attributes:
      localPreference: 200
      communities:
        standard: ["65001:service"]
```

#### IPAM Pool Configuration:

#### For Kubernetes:
```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumPodIPPool
metadata:
  name: pool-blue
spec:
  ipv4:
    cidrs:
    - 10.244.0.0/18
    maskSize: 24
  allocator: "cluster"
```

#### For OpenShift:
```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumPodIPPool
metadata:
  name: openshift-pool
spec:
  ipv4:
    cidrs:
    - 10.128.0.0/16  # OpenShift default pod CIDR
    maskSize: 23      # OpenShift default node allocation
  allocator: "cluster"
```

### 4. **Bridge Domain Configuration in ACI**
#### Detailed BD Setup:

#### For Kubernetes:
1. **Create Bridge Domain:**
   ```
   Name: k8s-pod-bd
   VRF: k8s-vrf
   Subnet: 10.244.0.0/16
   Scope: Public
   Subnet Control: Querier IP
   ```

#### For OpenShift:
1. **Create Bridge Domain:**
   ```
   Name: openshift-pod-bd
   VRF: k8s-vrf
   Subnet: 10.128.0.0/14
   Scope: Public
   Subnet Control: Querier IP
   ```

2. **Advanced BD Settings (Common):**
   ```
   ARP Flooding: Enabled
   Unknown Unicast: Proxy
   Unicast Routing: Enabled
   IGMP Snooping: Enabled
   MLD Snooping: Enabled (if IPv6)
   ```

3. **Associate with L3Out:**
   ```
   L3Out: k8s-l3out
   External EPG: k8s-external-epg
   Route Control: Import/Export
   ```

### 5. **Enterprise-Scale BGP Optimization**
#### Route Summarization:

#### For Kubernetes:
```bash
# Configure route aggregation in ACI
# APIC GUI: L3Out > Route Summarization
# Aggregate pod routes: 10.244.0.0/16
# Aggregate service routes: 10.96.0.0/12
```

#### For OpenShift:
```bash
# Configure route aggregation in ACI for OpenShift
# APIC GUI: L3Out > Route Summarization
# Aggregate pod routes: 10.128.0.0/14
# Aggregate service routes: 172.30.0.0/16
```

#### BGP Timer Optimization:
```yaml
# Optimized BGP timers for large scale
neighbors:
- peerAddress: 10.1.1.1/32
  peerASN: 65002
  holdTimeSeconds: 180    # Increased for stability
  keepaliveTimeSeconds: 60 # Increased interval
  connectRetryTimeSeconds: 30
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 300
```

#### Route Filtering and Policies:
```yaml
# Route filtering based on communities
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPAdvertisement
metadata:
  name: selective-advertisement
spec:
  advertisements:
  - advertisementType: "PodCIDR"
    attributes:
      communities:
        standard: ["65001:prod"]
      localPreference: 150
    selector:
      matchLabels:
        environment: "production"
```

## Advanced ACI L3Out Best Practices and Optimization

*Based on Cisco ACI L3Out White Paper recommendations*

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

#### ACI Health Monitoring:
- **APIC Monitoring**: Track BGP session state and route counts
- **Interface Utilization**: Monitor border leaf interface usage
- **Contract Hit Counters**: Verify security policy effectiveness
- **Fabric Health**: Check spine-leaf ISIS and MP-BGP sessions

### 6. **Security Integration**

#### ACI Contract Integration:
```yaml
# Cilium policies aligned with ACI contracts
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: aci-integrated-security
spec:
  endpointSelector:
    matchLabels:
      environment: "production"
  egress:
  - toCIDR:
    - "10.0.0.0/8"  # Match ACI external EPG scope
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
      - port: "80"
        protocol: TCP
  - toServices:
    - k8sService:
        serviceName: external-api
        namespace: production
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

## Advanced Kubernetes and OpenShift Node Network Configuration
### 1. **Enterprise-Grade Network Bonding Configuration**

#### Modern systemd-networkd Configuration (Recommended for both K8s and OpenShift):


#### 2. **Boot and BIOS Policies**
```yaml
Server Profile Template: k8s-node-template
  Boot Policy: k8s-uefi-boot
    Boot Mode: UEFI
    Secure Boot: Enabled
  
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
    VLAN: 200 (Node Network - BGP Peering)
    MTU: 9000
    MAC Pool: k8s-node-mac-pool
    Pin Group: k8s-node-pin-group
    QoS Policy: k8s-high-priority
    
  k8s-node-secondary:
    Fabric: B
    Redundancy Type: Secondary  
    VLAN: 200 (Node Network - BGP Peering)
    MTU: 9000
    MAC Pool: k8s-node-mac-pool
    Pin Group: k8s-node-pin-group
    QoS Policy: k8s-high-priority
```

#### 4. **QoS and Traffic Management**
```yaml
QoS Policy: k8s-high-priority
  Priority: Platinum
  Burst Size: 65536
  Rate Limit: 0 # Unlimited
  Host Control: Full
  
Network Control Policy: k8s-network-control
  CDP: Enabled
  LLDP Transmit: Enabled
  LLDP Receive: Enabled
  MAC Registration Mode: All Host VLANs
  Action on Uplink Fail: Link Down
  Forge MAC: Allow
```

### Detailed ACI Configuration for Kubernetes Integration

ACI configuration for Kubernetes integration requires careful planning of tenants, VRFs, bridge domains, and L3Outs. The following sections outline the manual configuration steps or can be automated using infrastructure as code tools.

#### Manual ACI Configuration Overview:

```yaml
Tenant Structure:
variable "apic_username" {
  description = "APIC Username"
  type        = string
}

variable "apic_password" {
  description = "APIC Password"
  type        = string
  sensitive   = true
}

variable "apic_url" {
  description = "APIC URL"
  type        = string
}

variable "tenant_name" {
  description = "ACI Tenant Name"
  type        = string
  default     = "K8s-Production"
}

# Create Tenant
resource "aci_tenant" "k8s_tenant" {
  name        = var.tenant_name
  description = "Kubernetes Production Tenant"
  
  annotation = "orchestrator:terraform"
}

# Create VRF
resource "aci_vrf" "k8s_vrf" {
  tenant_dn   = aci_tenant.k8s_tenant.id
  name        = "k8s-overlay-vrf"
  description = "Kubernetes Overlay VRF"
  
  pc_enf_pref = "enforced"
  pc_enf_dir  = "ingress"
}

# Create Bridge Domain for Node Network
resource "aci_bridge_domain" "k8s_nodes_bd" {
  tenant_dn          = aci_tenant.k8s_tenant.id
  name               = "k8s-nodes-bd"
  description        = "Kubernetes Nodes Bridge Domain"
  relation_fv_rs_ctx = aci_vrf.k8s_vrf.id
  
  mac                = "00:22:BD:F8:19:FF"
  arp_flood          = "yes"
  unicast_route      = "yes"
  unk_mac_ucast_act  = "proxy"
  unk_mcast_act      = "flood"
  multi_dst_pkt_act  = "bd-flood"
}

# Create Subnet for Node Network
resource "aci_subnet" "k8s_nodes_subnet" {
  parent_dn = aci_bridge_domain.k8s_nodes_bd.id
  ip        = "10.1.0.1/24"
  scope     = ["public", "shared"]
  description = "Kubernetes Nodes Subnet"
  
  ctrl = ["querier"]
  preferred = "yes"
}

# Create Bridge Domain for Kubernetes Pods
resource "aci_bridge_domain" "k8s_pods_bd" {
  tenant_dn          = aci_tenant.k8s_tenant.id
  name               = "k8s-pods-bd"
  description        = "Kubernetes Pods Bridge Domain"
  relation_fv_rs_ctx = aci_vrf.k8s_vrf.id
  
  mac                = "00:22:BD:F8:19:FE"
  arp_flood          = "yes"
  unicast_route      = "yes"
  unk_mac_ucast_act  = "proxy"
  unk_mcast_act      = "flood"
  multi_dst_pkt_act  = "bd-flood"
}

# Create Subnet for Kubernetes Pods
resource "aci_subnet" "k8s_pods_subnet" {
  parent_dn = aci_bridge_domain.k8s_pods_bd.id
  ip        = "10.244.0.1/16"
  scope     = ["public", "shared"]
  description = "Kubernetes Pods Subnet"
  
  ctrl = ["querier"]
  preferred = "yes"
}

# Create Bridge Domain for OpenShift Pods
resource "aci_bridge_domain" "openshift_pods_bd" {
  tenant_dn          = aci_tenant.k8s_tenant.id
  name               = "openshift-pods-bd"
  description        = "OpenShift Pods Bridge Domain"
  relation_fv_rs_ctx = aci_vrf.k8s_vrf.id
  
  mac                = "00:22:BD:F8:19:FD"
  arp_flood          = "yes"
  unicast_route      = "yes"
  unk_mac_ucast_act  = "proxy"
  unk_mcast_act      = "flood"
  multi_dst_pkt_act  = "bd-flood"
}

# Create Subnet for OpenShift Pods
resource "aci_subnet" "openshift_pods_subnet" {
  parent_dn = aci_bridge_domain.openshift_pods_bd.id
  ip        = "10.128.0.1/14"
  scope     = ["public", "shared"]
  description = "OpenShift Pods Subnet"
  
  ctrl = ["querier"]
  preferred = "yes"
}

# Create Bridge Domain for Kubernetes Services
resource "aci_bridge_domain" "k8s_services_bd" {
  tenant_dn          = aci_tenant.k8s_tenant.id
  name               = "k8s-services-bd"
  description        = "Kubernetes Services Bridge Domain"
  relation_fv_rs_ctx = aci_vrf.k8s_vrf.id
  
  mac                = "00:22:BD:F8:19:FC"
  arp_flood          = "yes"
  unicast_route      = "yes"
  unk_mac_ucast_act  = "proxy"
  unk_mcast_act      = "flood"
  multi_dst_pkt_act  = "bd-flood"
}

# Create Subnet for Kubernetes Services
resource "aci_subnet" "k8s_services_subnet" {
  parent_dn = aci_bridge_domain.k8s_services_bd.id
  ip        = "10.96.0.1/12"
  scope     = ["public", "shared"]
  description = "Kubernetes Services Subnet"
  
  ctrl = ["querier"]
  preferred = "yes"
}

# Create Bridge Domain for OpenShift Services
resource "aci_bridge_domain" "openshift_services_bd" {
  tenant_dn          = aci_tenant.k8s_tenant.id
  name               = "openshift-services-bd"
  description        = "OpenShift Services Bridge Domain"
  relation_fv_rs_ctx = aci_vrf.k8s_vrf.id
  
  mac                = "00:22:BD:F8:19:FB"
  arp_flood          = "yes"
  unicast_route      = "yes"
  unk_mac_ucast_act  = "proxy"
  unk_mcast_act      = "flood"
  multi_dst_pkt_act  = "bd-flood"
}

# Create Subnet for OpenShift Services
resource "aci_subnet" "openshift_services_subnet" {
  parent_dn = aci_bridge_domain.openshift_services_bd.id
  ip        = "172.30.0.1/16"
  scope     = ["public", "shared"]
  description = "OpenShift Services Subnet"
  
  ctrl = ["querier"]
  preferred = "yes"
}

# Create L3Out Domain
resource "aci_l3_domain_profile" "l3dom_external" {
  name        = "l3dom_external"
  description = "L3 Domain for external connectivity"
  
  annotation = "orchestrator:terraform"
}

# Create L3Out
resource "aci_l3_outside" "k8s_external_l3out" {
  tenant_dn      = aci_tenant.k8s_tenant.id
  name           = "k8s-external-l3out"
  description    = "Kubernetes External L3Out"
  relation_l3ext_rs_ectx = aci_vrf.k8s_vrf.id
  relation_l3ext_rs_l3_dom_att = aci_l3_domain_profile.l3dom_external.id
  
  target_dscp = "unspecified"
  annotation  = "orchestrator:terraform"
}

# Create Logical Node Profile
resource "aci_logical_node_profile" "k8s_node_profile" {
  l3_outside_dn = aci_l3_outside.k8s_external_l3out.id
  name          = "k8s-border-leafs"
  description   = "Logical node profile for border leafs"
  
  annotation = "orchestrator:terraform"
}

# Create Logical Node
resource "aci_logical_node_to_fabric_node" "leaf_101" {
  logical_node_profile_dn = aci_logical_node_profile.k8s_node_profile.id
  tdn                     = "topology/pod-1/node-101"
  rtr_id                  = "10.1.1.101"
  rtr_id_loop_back       = "yes"
  
  annotation = "orchestrator:terraform"
}

resource "aci_logical_node_to_fabric_node" "leaf_102" {
  logical_node_profile_dn = aci_logical_node_profile.k8s_node_profile.id
  tdn                     = "topology/pod-1/node-102"
  rtr_id                  = "10.1.1.102"
  rtr_id_loop_back       = "yes"
  
  annotation = "orchestrator:terraform"
}

# Create Logical Interface Profile
resource "aci_logical_interface_profile" "k8s_interface_profile" {
  logical_node_profile_dn = aci_logical_node_profile.k8s_node_profile.id
  name                    = "k8s-external-interfaces"
  description             = "Logical interface profile for external connectivity"
  
  annotation = "orchestrator:terraform"
}

# Create SVI Interface
resource "aci_l3out_path_attachment" "svi_interface_101" {
  logical_interface_profile_dn = aci_logical_interface_profile.k8s_interface_profile.id
  target_dn                    = "topology/pod-1/node-101/sys/ctx-[vxlan-2097152]/any/node-101/any/vlan-[vlan-100]"
  if_inst_t                    = "ext-svi"
  addr                         = "192.168.100.1/30"
  encap                        = "vlan-100"
  mode                         = "regular"
  mtu                          = "9000"
  
  annotation = "orchestrator:terraform"
}

resource "aci_l3out_path_attachment" "svi_interface_102" {
  logical_interface_profile_dn = aci_logical_interface_profile.k8s_interface_profile.id
  target_dn                    = "topology/pod-1/node-102/sys/ctx-[vxlan-2097152]/any/node-102/any/vlan-[vlan-100]"
  if_inst_t                    = "ext-svi"
  addr                         = "192.168.100.5/30"
  encap                        = "vlan-100"
  mode                         = "regular"
  mtu                          = "9000"
  
  annotation = "orchestrator:terraform"
}

# Create BGP Peer
resource "aci_bgp_peer_connectivity_profile" "external_bgp_peer_101" {
  logical_node_profile_dn = aci_logical_node_profile.k8s_node_profile.id
  addr                    = "192.168.100.2"
  addr_t_ctrl             = "af-ucast"
  allowed_self_as_cnt     = "3"
  annotation              = "orchestrator:terraform"
  as_number               = "65000"
  ctrl                    = ["send-com", "send-ext-com"]
  name_alias              = "external-router-101"
  peer_ctrl               = ["bfd"]
  private_a_sctrl         = ["remove-all", "remove-exclusive"]
  ttl                     = "1"
  weight                  = "0"
  
  local_as_number         = "65002"
  local_as_number_config  = "dual-as"
}

resource "aci_bgp_peer_connectivity_profile" "external_bgp_peer_102" {
  logical_node_profile_dn = aci_logical_node_profile.k8s_node_profile.id
  addr                    = "192.168.100.6"
  addr_t_ctrl             = "af-ucast"
  allowed_self_as_cnt     = "3"
  annotation              = "orchestrator:terraform"
  as_number               = "65000"
  ctrl                    = ["send-com", "send-ext-com"]
  name_alias              = "external-router-102"
  peer_ctrl               = ["bfd"]
  private_a_sctrl         = ["remove-all", "remove-exclusive"]
  ttl                     = "1"
  weight                  = "0"
  
  local_as_number         = "65002"
  local_as_number_config  = "dual-as"
}

# Create External EPG
resource "aci_external_network_instance_profile" "k8s_external_epg" {
  l3_outside_dn       = aci_l3_outside.k8s_external_l3out.id
  name                = "k8s-external-epg"
  description         = "External EPG for Kubernetes networks"
  exception_tag       = "0"
  flood_on_encap      = "disabled"
  match_t             = "AtleastOne"
  name_alias          = "k8s-ext-epg"
  pref_gr_memb        = "include"
  prio                = "unspecified"
  target_dscp         = "unspecified"
  
  annotation = "orchestrator:terraform"
}

# Create External Subnets for route control
resource "aci_l3_ext_subnet" "default_route" {
  external_network_instance_profile_dn = aci_external_network_instance_profile.k8s_external_epg.id
  ip                                   = "0.0.0.0/0"
  scope                               = ["import-security", "shared-security"]
  
  annotation = "orchestrator:terraform"
}

resource "aci_l3_ext_subnet" "k8s_pods_export" {
  external_network_instance_profile_dn = aci_external_network_instance_profile.k8s_external_epg.id
  ip                                   = "10.244.0.0/16"
  scope                               = ["export-rtctrl", "shared-rtctrl"]
  
  annotation = "orchestrator:terraform"
}

resource "aci_l3_ext_subnet" "k8s_services_export" {
  external_network_instance_profile_dn = aci_external_network_instance_profile.k8s_external_epg.id
  ip                                   = "10.96.0.0/12"
  scope                               = ["export-rtctrl", "shared-rtctrl"]
  
  annotation = "orchestrator:terraform"
}

resource "aci_l3_ext_subnet" "openshift_pods_export" {
  external_network_instance_profile_dn = aci_external_network_instance_profile.k8s_external_epg.id
  ip                                   = "10.128.0.0/14"
  scope                               = ["export-rtctrl", "shared-rtctrl"]
  
  annotation = "orchestrator:terraform"
}

resource "aci_l3_ext_subnet" "openshift_services_export" {
  external_network_instance_profile_dn = aci_external_network_instance_profile.k8s_external_epg.id
  ip                                   = "172.30.0.0/16"
  scope                               = ["export-rtctrl", "shared-rtctrl"]
  
  annotation = "orchestrator:terraform"
}

# Create Contracts
resource "aci_contract" "k8s_internal_contract" {
  tenant_dn   = aci_tenant.k8s_tenant.id
  name        = "k8s-internal-contract"
  description = "Internal communication contract for Kubernetes"
  scope       = "tenant"
  
  annotation = "orchestrator:terraform"
}

resource "aci_contract_subject" "k8s_internal_subject" {
  contract_dn   = aci_contract.k8s_internal_contract.id
  name          = "any-to-any"
  description   = "Allow any to any communication"
  cons_match_t  = "AtleastOne"
  prov_match_t  = "AtleastOne"
  prio          = "unspecified"
  rev_flt_ports = "yes"
  target_dscp   = "unspecified"
  
  annotation = "orchestrator:terraform"
}

# Create Filter for permit all
resource "aci_filter" "permit_all" {
  tenant_dn   = aci_tenant.k8s_tenant.id
  name        = "permit-all"
  description = "Permit all traffic filter"
  
  annotation = "orchestrator:terraform"
}

resource "aci_filter_entry" "permit_all_entry" {
  filter_dn   = aci_filter.permit_all.id
  name        = "permit-all"
  description = "Permit all traffic"
  ether_t     = "unspecified"
  prot        = "unspecified"
  
  annotation = "orchestrator:terraform"
}

# Associate filter with contract subject
resource "aci_contract_subject_filter" "k8s_internal_filter" {
  contract_subject_dn = aci_contract_subject.k8s_internal_subject.id
  filter_dn          = aci_filter.permit_all.id
  action             = "permit"
  directives         = ["none"]
  priority_override  = "default"
  
  annotation = "orchestrator:terraform"
}

# Output values
output "tenant_dn" {
  description = "Distinguished name of the created tenant"
  value       = aci_tenant.k8s_tenant.id
}

output "vrf_dn" {
  description = "Distinguished name of the created VRF"
  value       = aci_vrf.k8s_vrf.id
}

output "l3out_dn" {
  description = "Distinguished name of the created L3Out"
  value       = aci_l3_outside.k8s_external_l3out.id
}

output "external_epg_dn" {
  description = "Distinguished name of the external EPG"
  value       = aci_external_network_instance_profile.k8s_external_epg.id
}
```

**ACI Terraform Variables:**
```hcl
# terraform/aci/terraform.tfvars.example
apic_username = "admin"
apic_password = "your-password-here"
apic_url      = "https://apic.example.com"
tenant_name   = "K8s-Production"
```

**ACI Terraform Deployment:**
```bash
# Initialize and deploy ACI configuration
cd terraform/aci
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```
```yaml
Tenant Structure:
  Tenant: K8s-Production
    VRF: k8s-overlay-vrf
      Route Target: auto
      Policy Control Direction: ingress
      Policy Control Preference: enforced
      
    VRF: k8s-management-vrf  
      Route Target: auto
      Policy Control Direction: ingress
      
Bridge Domains:
  k8s-nodes-bd:
    VRF: k8s-overlay-vrf
    Subnet: 10.1.0.0/24
    Gateway: 10.1.0.1
    Scope: Public, Shared
    
  k8s-pods-bd:
    VRF: k8s-overlay-vrf  
    Subnet: 10.244.0.0/16  # Kubernetes default
    Gateway: 10.244.0.1
    Scope: Public, Shared
    
  openshift-pods-bd:
    VRF: k8s-overlay-vrf
    Subnet: 10.128.0.0/14  # OpenShift default
    Gateway: 10.128.0.1
    Scope: Public, Shared
    
  k8s-services-bd:
    VRF: k8s-overlay-vrf
    Subnet: 10.96.0.0/12   # Kubernetes services
    Gateway: 10.96.0.1
    Scope: Public, Shared
    
  openshift-services-bd:
    VRF: k8s-overlay-vrf
    Subnet: 172.30.0.0/16  # OpenShift services
    Gateway: 172.30.0.1
    Scope: Public, Shared
```

#### 2. **Advanced L3Out Configuration**
```yaml
L3Out: k8s-external-l3out
  VRF: k8s-overlay-vrf
  Domain: l3dom_external
  
  Logical Node Profile: k8s-border-leafs
    Nodes: leaf-101, leaf-102
    Router ID Loopback: Yes
    
  Logical Interface Profile: k8s-external-interfaces
    SVI Configuration:
      Interface: vlan-100
      IP: 192.168.100.1/30 (leaf-101)
      IP: 192.168.100.5/30 (leaf-102)  
      MTU: 9000
      
  BGP Peer Configuration:
    Peer IP: 192.168.100.2 (External Router A)
    Peer IP: 192.168.100.6 (External Router B)
    Remote ASN: 65000
    Local ASN: 65002
    
  External EPG: k8s-external-networks
    Subnets:
      - 0.0.0.0/0 (Default Route)
      - 10.244.0.0/16 (Kubernetes Pod Networks - Export)
      - 10.96.0.0/12 (Kubernetes Service Networks - Export)
      - 10.128.0.0/14 (OpenShift Pod Networks - Export)
      - 172.30.0.0/16 (OpenShift Service Networks - Export)
```

### Quality of Service and Traffic Engineering
#### 1. **QoS Configuration**
```yaml
QoS Classes:
  Level1 (Control Plane):
    Priority: 6
    Weight: 7
    BW Percent: 20
    
  Level2 (Data Plane):  
    Priority: 5
    Weight: 6
    BW Percent: 60
    
  Level3 (Best Effort):
    Priority: 1
    Weight: 1
    BW Percent: 20
```

#### 2. **Contract and Security Policy**
```yaml
Contracts:
  k8s-internal-contract:
    Subjects:
      - any-to-any:
          Filters: permit-all
          Direction: bidirectional
          
  k8s-external-contract:
    Subjects:
      - web-traffic:
          Filters: https, http
          Direction: bidirectional
      - ssh-management:
          Filters: ssh
          Direction: ingress
```

### Monitoring and Troubleshooting
#### 1. **APIC Monitoring Configuration**
```bash
# Enable fabric monitoring
apic1# configure
apic1(config)# monitoring policy default
apic1(config-monitoring)# collection interval 60
apic1(config-monitoring)# retention-policy 30days

# BGP session monitoring
apic1# show bgp ipv4 unicast summary vrf k8s-overlay-vrf
apic1# show bgp ipv4 unicast neighbors <peer-ip> vrf k8s-overlay-vrf
```

#### 2. **Performance Baselines**
```yaml
Key Metrics to Monitor:
  - BGP session state and route count
  - Leaf switch CPU and memory utilization  
  - Interface utilization and error rates
  - Contract hit counters
  - Endpoint learning and aging rates
  
Alerting Thresholds:
  - BGP session down: Immediate
  - Interface utilization > 80%: Warning
  - Error rate > 0.01%: Warning
  - Contract drops > 100/min: Critical
```

## Advanced Kubernetes and OpenShift Node Network Configuration
### 1. **Enterprise-Grade Network Bonding Configuration**

#### Modern systemd-networkd Configuration (Recommended for both K8s and OpenShift):
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
```

```bash
# /etc/systemd/network/20-bond0-ens3.network
[Match]
Name=ens3

[Network]
Bond=bond0
PrimarySlave=true
```

```bash
# /etc/systemd/network/21-bond0-ens4.network  
[Match]
Name=ens4

[Network]
Bond=bond0
```

```bash
# /etc/systemd/network/30-bond0.network
[Match]
Name=bond0

[Network]
DHCP=no
Address=10.1.0.10/24
Gateway=10.1.0.1
DNS=10.1.0.53
DNS=8.8.8.8

[Route]
Destination=10.244.0.0/16
Gateway=10.1.0.1
Metric=100
```

#### Legacy ifenslave Configuration (RHEL/CentOS for Kubernetes):
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

#### OpenShift RHCOS Network Configuration via Machine Config:
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
          Address=10.1.0.10/24
          Gateway=10.1.0.1
          DNS=10.1.0.53
          DNS=8.8.8.8
```

### 2. **SR-IOV Configuration for High Performance**
#### Enable SR-IOV in kernel:
```bash
# /etc/default/grub additions
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt pci=realloc"

# Update grub and reboot
update-grub && reboot

# Verify SR-IOV capability
lspci -v | grep -i sriov
```

#### SR-IOV VF Configuration:
```bash
# Enable VFs on physical interface
echo 8 > /sys/class/net/ens3/device/sriov_numvfs
echo 8 > /sys/class/net/ens4/device/sriov_numvfs

# Persistent VF configuration
cat > /etc/systemd/system/sriov-vfs.service << EOF
[Unit]
Description=Configure SR-IOV VFs
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo 8 > /sys/class/net/ens3/device/sriov_numvfs'
ExecStart=/bin/bash -c 'echo 8 > /sys/class/net/ens4/device/sriov_numvfs'

[Install]
WantedBy=multi-user.target
EOF

systemctl enable sriov-vfs.service
```

### 3. **Advanced Network Optimization**
#### Kernel Network Stack Tuning:
```bash
# /etc/sysctl.d/99-kubernetes-network.conf
# Core networking
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 5000
net.core.netdev_budget = 600
net.core.netdev_budget_usecs = 5000

# TCP optimization
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 90
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1

# Buffer sizes
net.core.rmem_default = 262144
net.core.rmem_max = 134217728
net.core.wmem_default = 262144
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 65536 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# IPv4 routing optimization
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.all.route_localnet = 1
net.ipv4.ip_nonlocal_bind = 1

# eBPF and netfilter optimization
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_buckets = 262144
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
```

#### IRQ Balancing and CPU Affinity:
```bash
# Install irqbalance
apt-get install irqbalance

# Configure irqbalance for NUMA awareness
cat > /etc/default/irqbalance << EOF
IRQBALANCE_ARGS="--policyscript=/usr/local/bin/irq-policy.sh"
EOF

# Custom IRQ policy script
cat > /usr/local/bin/irq-policy.sh << 'EOF'
#!/bin/bash
# Assign network IRQs to specific CPU cores
case "$1" in
    ens3|ens4)
        echo "package:0 cache:0"
        ;;
    *)
        echo "ignore"
        ;;
esac
EOF

chmod +x /usr/local/bin/irq-policy.sh
```

## Conclusion
Deploying Cilium with eBGP peering to Cisco ACI L3Outs represents a sophisticated approach to enterprise Kubernetes networking that bridges the gap between cloud-native workloads and traditional data center infrastructure. This comprehensive integration enables organizations to leverage the advanced networking capabilities of both platforms while maintaining the security, observability, and scalability requirements of modern enterprise environments.

### Key Technical Achievements
The implementation detailed in this document delivers several critical technical capabilities for both Kubernetes and OpenShift environments:

**Advanced Network Integration**: By establishing eBGP peering between Cilium's native BGP control plane and ACI L3Outs, organizations achieve seamless route advertisement and traffic engineering across both Kubernetes and OpenShift platforms. The use of local-only pod CIDR advertisements with /32 route optimization ensures scalability even in clusters with thousands of nodes while minimizing BGP convergence times and reducing route table sizes.

**Enterprise-Grade Security**: Cilium's eBPF-based policy enforcement combined with ACI's hardware-accelerated contract system provides defense-in-depth security at multiple layers. The integration supports L3-L7 policy enforcement, DNS-aware rules, and transparent encryption, enabling comprehensive microsegmentation and compliance with enterprise security frameworks. OpenShift's Security Context Constraints (SCCs) provide additional security layers that integrate seamlessly with Cilium's policy model.

**Platform-Specific Optimizations**: 
- **Kubernetes**: Full kube-proxy replacement with strict eBPF implementation for maximum performance
- **OpenShift**: Partial kube-proxy replacement ensuring compatibility with OpenShift's integrated monitoring and service mesh capabilities

**High-Performance Infrastructure**: The UCS B-Series integration with SR-IOV, active-backup bonding, and optimized network stack tuning delivers the performance characteristics required for demanding workloads on both platforms. The proper configuration of fabric interconnects, ACI leaf switches, and vPC ensures sub-second failover times and high availability.

### Operational Excellence and Observability
The Hubble integration provides unprecedented visibility into cluster networking behavior on both Kubernetes and OpenShift platforms, enabling proactive monitoring and rapid troubleshooting. Combined with ACI's comprehensive fabric monitoring and OpenShift's integrated monitoring stack, operators gain end-to-end observability from container networking through the physical infrastructure. OpenShift's native integration with Prometheus and Grafana enhances the monitoring capabilities with platform-specific dashboards and alerts.

### Scalability and Future-Proofing
The architecture scales horizontally through BGP route optimization and vertically through performance tuning. The modular design enables incremental adoption and integration with additional enterprise systems such as service mesh, storage networks, and external connectivity providers.

This implementation serves as a foundation for organizations seeking to modernize their data center networking while maintaining integration with existing Cisco infrastructure investments. The documented configurations and best practices ensure reliable, secure, and high-performance networking at enterprise scale for both traditional Kubernetes and enterprise OpenShift deployments. The flexibility to choose between platforms based on organizational needs while maintaining consistent networking and security policies represents a significant advancement in hybrid cloud infrastructure design.

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
