# microclos - 2x2 Layer 3 Clos Network Topology in GNS3 using MikroTik CHR

This repository contains the full architecture design, configuration blueprints, and empirical verification data for a 2-stage Layer 3 Clos (Leaf-Spine) network fabric simulated inside GNS3 using MikroTik Cloud Hosted Routers (CHR) running RouterOS v7.22.1.  The core objective of this deployment is to replace traditional Spanning Tree Protocol (STP) topologies with a fully routed backbone that implements Equal-Cost Multi-Pathing (ECMP) and Bidirectional Forwarding Detection (BFD) to achieve enterprise-grade, sub-second failover times.  🚀 Key Project Benefits100% Bandwidth Utilization: Zero links blocked by STP; traffic is actively distributed across all dual-spine paths using ECMP.  Sub-Second Convergence: Achieved an empirical failure recovery time of ~109ms by pairing OSPF with BFD.No Broadcast Storms: Completely isolates Layer 2 broadcast domains to individual leaf switches, removing core looping risks.  🗺️ Topology Design & IP AssignmentsThe fabric is constructed utilizing Point-to-Point /31 subnets (per RFC 3021) to maximize IP address efficiency.  Core Interconnects MapLinkDeviceInterfaceIP AddressOSPF Router IDRoleSpine01 ↔ Leaf01Spine-01Leaf-01ether1ether110.0.0.0/3110.0.0.1/311.1.1.1   Spine Router    Leaf Router   Spine01 ↔ Leaf02Spine-01Leaf-02ether2ether110.0.0.2/3110.0.0.3/311.1.1.1   Spine Router    Leaf Router   Spine02 ↔ Leaf01Spine-02Leaf-01ether1ether210.0.0.4/3110.0.0.5/312.2.2.2   Spine Router    Leaf Router   Spine02 ↔ Leaf02Spine-02Leaf-02ether2ether210.0.0.6/3110.0.0.7/312.2.2.2   Spine Router    Leaf Router   Downstream Access NetworksLeaf-01 (Debian Server Segment): 172.16.0.254/24 tied to bridge1.  Leaf-02 (VPCS Client Segment): 192.168.1.254/24 tied to bridge1.  🛠️ Configuration Blueprints1. Spine Configuration (Spine-01 Example)Code snippet# IP Addressing
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
(Repeat symmetrically for Spine-02 using Router ID 2.2.2.2 and its corresponding /31 IPs).  2. Leaf Configuration (Leaf-01 Example)To prevent network loops while keeping access-layer availability, access-facing interfaces use local bridges, but uplink interfaces are left out of the bridge to enforce Layer 3 routing.  Code snippet# Uplink IP Addressing
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
3. Leaf-02 DHCP Relay SetupTo allow clients sitting behind Leaf-02 to pull dynamic configurations across the routed core from the centralized server behind Leaf-01, a standard DHCP relay is bound to the client interface:  Code snippet/interface bridge add name=bridge1
/interface bridge port add interface=ether8 bridge=bridge1
/ip address add address=192.168.1.254/24 interface=bridge1

/ip dhcp-relay add name=relay1 interface=bridge1 dhcp-server=172.16.0.1 local-address=192.168.1.254 disabled=no
📊 Operational Verification & Convergence TestingRun these troubleshooting commands in the RouterOS terminal to ensure the fabric is healthy:1. Active OSPF AdjacenciesVerify that all links have established a Full state connection:  Code snippet/routing ospf neighbor print
Expected Output:PlaintextFlags: D - DYNAMIC
0 D instance=ospf1 area=backbone interface=ether1 address=10.0.0.5 router-id=3.3.3.3 state=Full
1 D instance=ospf1 area=backbone interface=ether2 address=10.0.0.7 router-id=4.4.4.4 state=Full
2. Active BFD SessionsEnsure that the low-overhead hardware-tracked keepalives are actively up:  Code snippet/routing bfd session print
Expected Output:PlaintextFlags: U - UP
0 U multihop=no vrf=main remote-address=10.0.0.2%ether1 state=up actual-tx-interval=100ms hold-time=300ms
1 U multihop=no vrf=main remote-address=10.0.0.6%ether2 state=up actual-tx-interval=100ms hold-time=300ms
3. Routing Paths & Active ECMP Multi-PathingConfirm that your traffic load-balances over both Spine choices simultaneously via the routing matrix (+ flag indicates active ECMP):  Code snippet/ip route print
Expected Output Snippet:PlaintextDAo+ 172.16.0.254/32  10.0.0.2%ether1  main 110
DAo+ 172.16.0.254/32  10.0.0.6%ether2  main 110
📉 Empirical Convergence BenchmarksDuring a physical link interruption stress test (monitored via a continuous 100ms ping run alongside Wireshark capture traces), the fabric achieved the following failover markers:Default OSPF (No BFD): ~40 Seconds OSPF Aggressive Timers (No BFD): ~3 Seconds OSPF + BFD (This Lab Design): 109 Milliseconds (Only 1 single packet dropped before full alternate-path restabilization).🧠 Key Architecture TakeawaysDevice Optimization Choice: Initial deployment concepts using physical edge switches (like the MikroTik CRS328) were rejected because RouterOS switch packages omit full sub-second BFD engine features. Simulating with MikroTik CHR instances successfully brings full core-routing architecture advantages directly down to small LAN environments.  Horizontal Scaling Paths: Scaling out aggregate network capacity in a 2-stage Clos structure simply requires dropping in an additional Spine node; scaling port access is completed by attaching another standalone Leaf.
