# Deploying ACI CNI with eBGP Peering to Cisco ACI L3Outs: A Comprehensive Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [ACI CNI Overview](#aci-cni-overview)
4. [Kubernetes Cluster Preparation](#kubernetes-cluster-preparation)
5. [Installing ACI CNI on Kubernetes](#installing-aci-cni-on-kubernetes)
6. [Configuring eBGP Peering with ACI L3Outs](#configuring-ebgp-peering-with-aci-l3outs)
    - [Advertising Pod and Service CIDRs](#advertising-pod-and-service-cidrs)
    - [Bridge Domain for Node/Compute Network](#bridge-domain-for-nodecompute-network)
    - [Scale Configurations and /32 Route Optimization](#scale-configurations-and-32-route-optimization)
7. [Network Policies and Security with ACI CNI](#network-policies-and-security-with-aci-cni)
8. [UCS B-Series Configuration](#ucs-b-series-configuration)
9. [Connecting to Upstream ACI Fabric](#connecting-to-upstream-aci-fabric)
10. [Kubernetes Node Network Configuration (Active/Backup)](#kubernetes-node-network-configuration-activebackup)
11. [Conclusion](#conclusion)
12. [References](#references)

---

## Introduction
This document provides a comprehensive, thesis-level guide to deploying Cisco ACI CNI on Kubernetes, establishing eBGP peering with ACI L3Outs, advertising pod and service CIDRs, and integrating with UCS B-Series and ACI fabric. It covers advanced topics such as scale optimization, network security, and high-availability node networking.

## Prerequisites
- Kubernetes cluster (v1.23+ recommended)
- Access to Cisco ACI fabric with L3Outs configured
- UCS B-Series servers (if applicable)
- Administrative access to all systems
- Familiarity with BGP, Kubernetes networking, and ACI concepts

## ACI CNI Overview
Cisco ACI CNI is a Kubernetes CNI plugin that integrates Kubernetes networking directly with Cisco ACI fabric, providing:
- Native pod networking mapped to ACI bridge domains and EPGs
- Automated endpoint learning and policy enforcement
- Support for BGP (including eBGP peering via L3Outs)
- Centralized security and microsegmentation via ACI contracts
- High scalability and performance

## Kubernetes Cluster Preparation
1. **Provision Nodes:**
   - Deploy Kubernetes nodes (bare metal or VMs) on UCS B-Series or other compute platforms.
   - Ensure nodes are connected to the ACI fabric via appropriate VLANs/bridge domains.
2. **Networking:**
   - Assign static or DHCP IPs to nodes, ideally from a dedicated compute subnet/bridge domain.
   - Ensure L3Out connectivity from nodes to ACI fabric.
3. **High Availability:**
   - Configure nodes with dual NICs (active/backup) for redundancy.
   - Use Linux bonding (mode=active-backup) or similar.

## Installing ACI CNI on Kubernetes
1. **Download and Prepare ACI CNI Installer:**
   - Obtain the ACI CNI installer bundle from Cisco (requires CCO login).
   - Extract the installer and review the `aci-containers-config.yaml` file.
2. **Configure ACI CNI Parameters:**
   - Set APIC URL, credentials, ACI tenant, VRF, bridge domains, EPGs, and L3Outs in the config file.
   - Define pod and service CIDRs to match your cluster design.
   - Enable BGP peering and specify ASN and peer IPs as needed.
3. **Install ACI CNI:**
   ```bash
   kubectl apply -f aci-containers-config.yaml
   kubectl apply -f aci-containers.yaml
   ```
   - Monitor pods in the `aci-containers-system` namespace for readiness.

## Configuring eBGP Peering with ACI L3Outs
### 1. **ACI L3Out Preparation**
- Create an L3Out in ACI for the Kubernetes compute network.
- Assign a unique ASN to the ACI L3Out (e.g., 65002).
- Configure BGP peerings to each Kubernetes node IP (or loopback if using /32s).
- Ensure the L3Out EPG is associated with the correct bridge domain.

### 2. **ACI CNI BGP Configuration**
- In `aci-containers-config.yaml`, configure the BGP section:
```yaml
bgp:
  asn: 65001
  peers:
    - address: <ACI_L3OUT_PEER_IP>
      asn: 65002
      ebgpMultihop: 2
      holdTime: 90
      keepaliveTime: 30
      password: <optional>
  advertisePodCIDR: true
  advertiseServiceCIDR: true
  advertiseNodeIP: true
  localPodCIDROnly: true  # Only advertise local pod CIDRs as /32s for scale
```
- Replace `<ACI_L3OUT_PEER_IP>` with the ACI L3Out SVI or loopback IP.
- `localPodCIDROnly: true` ensures only the local node's /32 is advertised, optimizing scale.

### 3. **Advertising Pod and Service CIDRs**
- ACI CNI can advertise both pod and service CIDRs via BGP.
- Ensure `advertisePodCIDR` and `advertiseServiceCIDR` are enabled.
- For large clusters, use `localPodCIDROnly: true` to advertise only the local node's podCIDR as a /32 route, reducing BGP table size.

### 4. **Bridge Domain for Node/Compute Network**
- In ACI, create a bridge domain for the Kubernetes node subnet.
- Associate the bridge domain with the L3Out EPG.
- Enable ARP flooding and unicast routing as needed.

### 5. **Scale Configurations and /32 Route Optimization**
- For clusters with hundreds or thousands of nodes, advertising only /32s (node IPs) and local podCIDRs reduces BGP churn and table size.
- Use `localPodCIDROnly: true` in ACI CNI config to limit advertisements to local resources.
- Monitor BGP session counts and route table size in ACI.

## Network Policies and Security with ACI CNI
- ACI CNI leverages ACI contracts for network policy enforcement.
- Define Kubernetes NetworkPolicies, which are translated to ACI contracts and enforced in hardware.
- Example: Allow only HTTP traffic to a namespace
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
  namespace: my-namespace
spec:
  podSelector: {}
  ingress:
  - ports:
    - protocol: TCP
      port: 80
```
- Use ACI contracts for microsegmentation between EPGs, pods, and external networks.
- Integrate with ACI monitoring tools for visibility and auditing.

## UCS B-Series Configuration
- Ensure UCS blades are connected to ACI leaf switches via Fabric Interconnects.
- Configure vNICs for each blade, mapped to the correct VLANs (Kubernetes node, pod, and service networks).
- Use Service Profiles for consistent network and firmware settings.
- Enable LACP or active/backup NIC teaming for redundancy.
- Ensure BIOS and firmware are up to date for optimal performance.

## Connecting to Upstream ACI Fabric
- Connect UCS Fabric Interconnect uplinks to ACI leaf switches.
- Ensure VLANs for Kubernetes node and pod networks are allowed on uplinks.
- Configure ACI EPGs and bridge domains for each network segment.
- Use static path bindings for precise control.
- Validate end-to-end connectivity from Kubernetes nodes to ACI L3Outs.

## Kubernetes Node Network Configuration (Active/Backup)
- Configure Linux bonding on each node:
  ```bash
  # /etc/netplan/01-bond.yaml (Ubuntu example)
  network:
    version: 2
    bonds:
      bond0:
        interfaces: [eth0, eth1]
        parameters:
          mode: active-backup
          primary: eth0
        addresses: [10.1.1.10/24]
        gateway4: 10.1.1.1
        nameservers:
          addresses: [8.8.8.8,8.8.4.4]
  ```
- Ensure both NICs are cabled to different UCS FIs and ACI leafs for redundancy.
- Validate failover by disconnecting one link at a time.

## Conclusion
Deploying ACI CNI with eBGP peering to Cisco ACI L3Outs enables scalable, secure, and observable Kubernetes networking. By leveraging ACI's centralized policy model, optimized BGP route advertisements, and robust hardware enforcement, organizations can achieve seamless integration between cloud-native workloads and enterprise networks. Proper configuration of UCS, ACI, and Kubernetes nodes ensures high availability and performance at scale.

## References
- [ACI CNI Documentation](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/aci-containers/)
- [Cisco ACI BGP L3Out Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/4-x/l3out/b_Cisco_APIC_Layer_3_Outside_Configuration_Guide.html)
- [UCS B-Series Configuration Guides](https://www.cisco.com/c/en/us/support/servers-unified-computing/ucs-b-series-blade-servers/products-installation-guides-list.html)
- [Kubernetes Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
