# Router-on-a-Stick with VTP, EtherChannel & Inter-Router DHCP Relay

## Objective
Configure a multi-VLAN network using VTP for centralized VLAN propagation, 
EtherChannel for redundant inter-switch links, and Router-on-a-Stick for 
inter-VLAN routing — with DHCP served from a centralized server across a 
second router via a point-to-point routed link.

## Topology
![Topology](topology.png)

- **Router0 (ISR4331)** — subinterfaces for VLAN 10/20/30, Router-on-a-Stick config, DHCP relay
- **Router1 (ISR4331)** — connects Router0 to the DHCP server subnet
- **Switch0** — VTP Server, EtherChannel to Switch1, trunk to Router0
- **Switch1** — VTP Client, EtherChannel to Switch0, trunk to Switch2
- **Switch2** — VTP Client, trunk to Switch1
- **Server0** — DHCP pools for client address assignment

## Addressing
| Interface / Link           | IP Address    |
|-----------------------------|--------------|
| Router0 Gig0/0/1            | 192.168.4.1  |
| Router1 Gig0/0/0            | 192.168.4.2  |
| Router1 Gig0/0/1            | 192.168.5.1  |
| Server0 (DHCP Pool)         | 192.168.5.2  |
| VLAN 10 Gateway (Fa0/1.10)  | 192.168.1.1  |
| VLAN 20 Gateway (Fa0/1.20)  | 192.168.2.1  |
| VLAN 30 Gateway (Fa0/1.30)  | 192.168.3.1  |

## VLANs
| VLAN | Assigned On      | Hosts     |
|------|-------------------|-----------|
| 10   | Switch0            | PC0, PC1  |
| 20   | Switch0, Switch2   | PC2, PC3  |
| 30   | Switch1            | PC4, PC5  |

## Configuration Steps

**1. VTP & VLAN Setup — Switch0 (VTP Server)**
- Configured Switch0 as the VTP server, set the VTP domain name and password
- Created VLANs 10, 20, and 30 with descriptive names
- Assigned VLAN 10 and VLAN 20 to their respective access ports
- Configured the trunk link toward Router0 for Router-on-a-Stick
- Bundled the links to Switch1 into an EtherChannel

**2. EtherChannel & VTP Client — Switch1**
- Configured the matching EtherChannel on Switch1's side to form the port-channel with Switch0
- Set Switch1 to VTP client mode with the matching domain and password to sync VLAN info
- Assigned VLAN 30 to its access port
- Configured a trunk link down to Switch2

**3. VTP Client & Trunking — Switch2**
- Configured the trunk link upward to Switch1
- Set Switch2 to VTP client mode with the matching domain and password
- Assigned VLAN 20 to its access port

**4. Inter-VLAN Routing & DHCP Relay — Routers**
- Configured Router-on-a-Stick on Router0 with subinterfaces for each VLAN and assigned gateway IPs
- Added `ip helper-address` on each VLAN subinterface, pointing to the DHCP server to relay client requests across the routed link
- Repeated subinterface and helper-address configuration on Router1

**5. DHCP Server**
- Created DHCP address pools on Server0 for each VLAN subnet and enabled the DHCP service

## Key Configuration
