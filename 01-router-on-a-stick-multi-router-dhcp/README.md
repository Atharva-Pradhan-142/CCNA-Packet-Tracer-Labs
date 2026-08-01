# Router-on-a-Stick with VTP, EtherChannel & Inter-Router DHCP Relay

## Objective
Configure a multi-VLAN network using VTP for centralized VLAN propagation, 
EtherChannel for redundant inter-switch links, and Router-on-a-Stick for 
inter-VLAN routing — with DHCP served from a centralized server across a second router, using OSPF for dynamic routing between Router0 and Router1.

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
| VLAN 10 Gateway (gig0/0/0.10)  | 192.168.1.1  |
| VLAN 20 Gateway (gig0/0/0.20)  | 192.168.2.1  |
| VLAN 30 Gateway (gig0/0/0.30)  | 192.168.3.1  |

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

### Switch0 (VTP Server + EtherChannel to Switch1)

vtp domain abc.com 
vtp password 123
vtp mode server

vlan 10
name IT
vlan 20
name HR
vlan 30
name Sales

interface range Fa0/2-3
switchport mode access
switchport access vlan 10

interface Fa0/4
switchport mode access
switchport access vlan 20

interface Fa0/1
switchport mode trunk

interface range Fa0/5-6
channel-group 1 mode active

interface Port-channel 1
switchport mode trunk

### Switch1 (VTP Client + EtherChannel to Switch0)

interface range Fa0/1-2
channel-group 1 mode active

interface Port-channel 1
switchport mode trunk

vtp mode client
vtp domain abc.com
vtp password 123

interface range Fa0/4-5
switchport mode access
switchport access vlan 30

interface Fa0/3
switchport mode trunk

### Switch2 (VTP Client)

interface Fa0/2
switchport mode trunk

vtp mode client
vtp domain abc.com
vtp password 123

interface Fa0/1
switchport mode access
switchport access vlan 20

### Router0 (Router-on-a-Stick + DHCP Relay + OSPF)

interface GigabitEthernet0/0/1
ip address 192.168.4.1 255.255.255.0
no shutdown

interface GigabitEthernet0/0/0
no shutdown

interface GigabitEthernet0/0/0.10
encapsulation dot1Q 10
ip address 192.168.1.1 255.255.255.0
ip helper-address 192.168.5.2

interface GigabitEthernet0/0/0.20
encapsulation dot1Q 20
ip address 192.168.2.1 255.255.255.0
ip helper-address 192.168.5.2

interface GigabitEthernet0/0/0.30
encapsulation dot1Q 30
ip address 192.168.3.1 255.255.255.0
ip helper-address 192.168.5.2

router ospf 1
network 192.168.1.0 0.0.0.255 area 0
network 192.168.2.0 0.0.0.255 area 0
network 192.168.3.0 0.0.0.255 area 0
network 192.168.4.0 0.0.0.255 area 0

### Router1 (WAN Link to Router0 + OSPF)

interface GigabitEthernet0/0/1
ip address 192.168.5.1 255.255.255.0
no shutdown

interface GigabitEthernet0/0/0
ip address 192.168.4.2 255.255.255.0
no shutdown

router ospf 1
network 192.168.4.0 0.0.0.255 area 0
network 192.168.5.0 0.0.0.255 area 0

## Verification
- [x] VTP propagated VLANs from Switch0 to Switch1 and Switch2 (`show vtp status`)
- [x] EtherChannel formed successfully between Switch0 and Switch1 (`show etherchannel summary`)
- [x] All VLANs can ping their gateway
- [x] Inter-VLAN routing confirmed (VLAN 10 ↔ VLAN 20 ↔ VLAN 30)
- [x] DHCP clients successfully lease IPs from Server0 across the Router0–Router1 link
- [x] End-to-end reachability from VLAN hosts to Server0

## Skills Demonstrated
VTP (server/client modes), VLAN propagation, EtherChannel/port-channel 
configuration, 802.1Q trunking, Router-on-a-Stick inter-VLAN routing, 
DHCP relay via `ip helper-address`, centralized DHCP server configuration, 
OSPF dynamic routing, LACP EtherChannel
