# Microclos - 2x2 Layer 3 Clos Network Topology in GNS3 using MikroTik CHR

This repository contains the full architecture design, configuration blueprints, and empirical verification data for a 2-stage Layer 3 Clos (Leaf-Spine) network fabric simulated inside GNS3 using MikroTik Cloud Hosted Routers (CHR) running RouterOS v7.22.1.

The core objective of this deployment is to replace traditional Spanning Tree Protocol (STP) topologies with a fully routed backbone that implements Equal-Cost Multi-Pathing (ECMP) and Bidirectional Forwarding Detection (BFD) to achieve enterprise-grade, sub-second failover times.

---

## Key Project Benefits
* 100% Bandwidth Utilization: Zero links blocked by STP; traffic is actively distributed across all dual-spine paths using ECMP.
* Sub-Second Convergence: Achieved an empirical failure recovery time of ~109ms by pairing OSPF with BFD.
* No Broadcast Storms: Completely isolates Layer 2 broadcast domains to individual leaf switches, removing core looping risks.

---

## 🗺️ Topology Design & IP Assignments

The fabric is constructed utilizing Point-to-Point /31 subnets (per RFC 3021) to maximize IP address efficiency.

### Core Interconnects Map

| Link | Device | Interface | IP Address | OSPF Router ID | Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Spine01 <-> Leaf01 | Spine-01 <br> Leaf-01 | ether1 <br> ether1 | 10.0.0.0/31 <br> 10.0.0.1/31 | 1.1.1.1 | Spine Router <br> Leaf Router |
| Spine01 <-> Leaf02 | Spine-01 <br> Leaf-02 | ether2 <br> ether1 | 10.0.0.2/31 <br> 10.0.0.3/31 | 1.1.1.1 | Spine Router <br> Leaf Router |
| Spine02 <-> Leaf01 | Spine-02 <br> Leaf-01 | ether1 <br> ether2 | 10.0.0.4/31 <br> 10.0.0.5/31 | 2.2.2.2 | Spine Router <br> Leaf Router |
| Spine02 <-> Leaf02 | Spine-02 <br> Leaf-02 | ether2 <br> ether2 | 10.0.0.6/31 <br> 10.0.0.7/31 | 2.2.2.2 | Spine Router <br> Leaf Router |

### Downstream Access Networks
* Leaf-01 (Debian Server Segment): 172.16.0.254/24 tied to bridge1.
* Leaf-02 (VPCS Client Segment): 192.168.1.254/24 tied to bridge1.

---

## 🛠️ Configuration Blueprints

### 1. Spine Configuration (Spine-01 Example)

# IP Addressing
/ip address add address=10.0.0.0/31 interface=ether1
/ip address add address=10.0.0.2/31 interface=ether2

# OSPF Instance & Backbone Area
/routing ospf instance add name=ospf1 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1

# OSPF Templates (Point-to-Point)
/routing ospf interface-template add interfaces=ether1 area=backbone type=ptp hello-interval=1s dead-interval=3s use-bfd=yes
/routing ospf interface-template add interfaces=ether2 area=backbone type=ptp hello-interval=1s dead-interval=3s use-bfd=yes

# BFD Custom Micro-timers
/routing bfd configuration add interfaces=ether1 min-rx=100ms min-tx=100ms multiplier=3
/routing bfd configuration add interfaces=ether2 min-rx=100ms min-tx=100ms multiplier=3


### 2. Leaf Configuration (Leaf-01 Example)

# Uplink IP Addressing
/ip address add address=10.0.0.1/31 interface=ether1
/ip address add address=10.0.0.5/31 interface=ether2

# Access Segment Bridge Setup
/interface bridge add name=bridge1
/interface bridge port add interface=ether8 bridge=bridge1
/ip address add address=172.16.0.254/24 interface=bridge1

# OSPF Setup
/routing ospf instance add name=ospf1 router-id=3.3.3.3
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1

# OSPF Templates (Active Uplinks + Passive Bridge)
/routing ospf interface-template add interfaces=ether1 area=backbone type=ptp hello-interval=1s dead-interval=3s use-bfd=yes
/routing ospf interface-template add interfaces=ether2 area=backbone type=ptp hello-interval=1s dead-interval=3s use-bfd=yes
/routing ospf interface-template add interfaces=bridge1 area=backbone type=ptp passive=yes

# BFD Configuration
/routing bfd configuration add interfaces=ether1 min-rx=100ms min-tx=100ms multiplier=3
/routing bfd configuration add interfaces=ether2 min-rx=100ms min-tx=100ms multiplier=3


### 3. Leaf-02 DHCP Relay Setup

/interface bridge add name=bridge1
/interface bridge port add interface=ether8 bridge=bridge1
/ip address add address=192.168.1.254/24 interface=bridge1

/ip dhcp-relay add name=relay1 interface=bridge1 dhcp-server=172.16.0.1 local-address=192.168.1.254 disabled=no

---

## 📊 Operational Verification & Convergence Testing

### 1. Active OSPF Adjacencies

Command:
/routing ospf neighbor print

Expected CLI Output:
Flags: D - DYNAMIC
0 D instance=ospf1 area=backbone interface=ether1 address=10.0.0.5 router-id=3.3.3.3 state=Full
1 D instance=ospf1 area=backbone interface=ether2 address=10.0.0.7 router-id=4.4.4.4 state=Full


### 2. Active BFD Sessions

Command:
/routing bfd session print

Expected CLI Output:
Flags: U - UP
0 U multihop=no vrf=main remote-address=10.0.0.2%ether1 state=up actual-tx-interval=100ms hold-time=300ms
1 U multihop=no vrf=main remote-address=10.0.0.6%ether2 state=up actual-tx-interval=100ms hold-time=300ms


### 3. Routing Paths & Active ECMP Multi-Pathing

Command:
/ip route print

Expected CLI Output:
DAo+ 172.16.0.254/32  10.0.0.2%ether1  main 110
DAo+ 172.16.0.254/32  10.0.0.6%ether2  main 110


### Empirical Convergence Benchmarks
* Default OSPF (No BFD): ~40 Seconds
* OSPF Aggressive Timers (No BFD): ~3 Seconds
* OSPF + BFD (This Lab Design): 109 Milliseconds (Only 1 single packet dropped before full alternate-path restabilization).

---

### Diagram of the Topology

![Network Topology Diagram](topology.png)

## Key Architecture Takeaways
1. Device Optimization Choice: Initial deployment concepts using physical edge switches (like the MikroTik CRS328) were rejected because RouterOS switch packages omit full sub-second BFD engine features. Simulating with MikroTik CHR instances successfully brings full core-routing architecture advantages directly down to small LAN environments.
2. Horizontal Scaling Paths: Scaling out aggregate network capacity in a 2-stage Clos structure simply requires dropping in an additional Spine node; scaling port access is completed by attaching another standalone Leaf.
