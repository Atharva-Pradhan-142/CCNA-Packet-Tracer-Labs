# Secure Branch Office Network — Router-on-a-Stick, OSPF, NAT Overload, ACL Filtering, AAA (TACACS+) & Port Security

## Objective
Build a two-site network (Branch Office + Head Office) connected over a
serial WAN link running OSPF. The Branch router performs Router-on-a-Stick
routing for five VLANs, provides per-VLAN DHCP, and overloads all branch
traffic through NAT/PAT toward the Head Office. Extended ACLs enforce
department-level access to the server VLANs, AAA (TACACS+ with local
fallback) and SSH-only access secure the Branch router, and the access-layer
switches run DHCP Snooping, Dynamic ARP Inspection, and Port Security to
protect the edge.

## Topology
![Topology](topology.png)

- **R1 (ISR4331, Branch)** — Router-on-a-Stick for VLANs 10/20/30/40/50,
  per-VLAN DHCP server, NAT overload (PAT) to Head Office, OSPF, extended
  ACLs on the HR and Sales sub-interfaces, AAA login via TACACS+ (local
  fallback), SSH-only VTY access
- **SW-Core (below R1)** — trunk to R1 plus trunk uplinks to three
  distribution switches; DHCP Snooping and Dynamic ARP Inspection enabled
  on all VLANs, trunk ports trusted, access ports rate-limited
- **SW-Left** — access switch for VLAN 10/IT (PC0) and VLAN 20/HR (PC1)
- **SW-Bottom** — access switch for VLAN 30/Sales (PC2, PC3)
- **SW-Right** — access switch for VLAN 40/Management (PC4) and VLAN
  50/Servers (Server1), with Port Security (sticky MAC) on the server port
- **R2 (ISR4331, Head Office)** — routes between the WAN link and the
  server VLAN, NAT overload for Head Office traffic, OSPF
- **SW-Server (below R2)** — trunk to R2, access port for Server0
- **Server0** — Head Office server, `10.10.10.10/24`
- **Server1** — Branch-local server on VLAN 50, port-secured
- **TACACS+ server** — `192.168.100.146`, used for AAA on R1

## VLSM Addressing Scheme (192.168.100.0/24 — Branch)

| Subnet          | Mask              | CIDR | Usable Range | Assigned To                    |
|------------------|-------------------|------|----------------|-----------------------------------|
| 192.168.100.0    | 255.255.255.192   | /26  | .1 – .62       | VLAN 30 — Sales                  |
| 192.168.100.64   | 255.255.255.224   | /27  | .65 – .94      | VLAN 20 — HR                     |
| 192.168.100.96   | 255.255.255.224   | /27  | .97 – .126     | VLAN 10 — IT                     |
| 192.168.100.128  | 255.255.255.240   | /28  | .129 – .142    | VLAN 40 — Management             |
| 192.168.100.144  | 255.255.255.248   | /29  | .145 – .150    | VLAN 50 — Servers (Server1)      |

### WAN & Head Office

| Subnet        | Mask              | CIDR | Assigned To                       |
|----------------|-------------------|------|------------------------------------|
| 172.16.1.0     | 255.255.255.252   | /30  | R1 ↔ R2 WAN (Serial0/1/0)         |
| 10.10.10.0     | 255.255.255.0     | /24  | Head Office LAN (Server0)         |

## VLANs (Branch)

| VLAN | Gateway              | Hosts        | Access Policy                                   |
|------|------------------------|--------------|--------------------------------------------------|
| 10   | 192.168.100.97/27      | PC0 (IT)     | Unrestricted                                     |
| 20   | 192.168.100.65/27      | PC1 (HR)     | Denied to both server subnets (ACL 110)          |
| 30   | 192.168.100.1/26       | PC2, PC3 (Sales) | HTTP/HTTPS/ICMP only to server subnets (ACL 120) |
| 40   | 192.168.100.129/28     | PC4 (Management) | Unrestricted (full access)                    |
| 50   | 192.168.100.145/29     | Server1      | Port-secured, no inbound ACL                     |

## Key Configuration

### R1 (Branch — Router-on-a-Stick, DHCP, NAT, OSPF, AAA, SSH)

```
hostname h1
enable password 123

aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
username shhaccess privilege 15 password 0 ssh
ip domain-name abc.com

tacacs-server host 192.168.100.146
tacacs-server key secret

ip dhcp pool 10
 network 192.168.100.96 255.255.255.224
 default-router 192.168.100.97
ip dhcp pool 20
 network 192.168.100.64 255.255.255.224
 default-router 192.168.100.65
ip dhcp pool 30
 network 192.168.100.0 255.255.255.192
 default-router 192.168.100.1
ip dhcp pool 40
 network 192.168.100.128 255.255.255.240
 default-router 192.168.100.129
ip dhcp pool 50
 network 192.168.100.144 255.255.255.248
 default-router 192.168.100.145

interface GigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.100.97 255.255.255.224
 ip nat inside

interface GigabitEthernet0/0/0.20
 encapsulation dot1Q 20
 ip address 192.168.100.65 255.255.255.224
 ip nat inside
 ip access-group 110 in

interface GigabitEthernet0/0/0.30
 encapsulation dot1Q 30
 ip address 192.168.100.1 255.255.255.192
 ip nat inside
 ip access-group 120 in

interface GigabitEthernet0/0/0.40
 encapsulation dot1Q 40
 ip address 192.168.100.129 255.255.255.240
 ip nat inside

interface GigabitEthernet0/0/0.50
 encapsulation dot1Q 50
 ip address 192.168.100.145 255.255.255.248
 ip nat inside

interface Serial0/1/0
 ip address 172.16.1.1 255.255.255.252
 ip nat outside
 clock rate 2000000

router ospf 1
 network 192.168.100.0 0.0.0.63 area 0
 network 192.168.100.64 0.0.0.31 area 0
 network 192.168.100.96 0.0.0.31 area 0
 network 192.168.100.128 0.0.0.15 area 0
 network 192.168.100.144 0.0.0.7 area 0
 network 172.16.1.0 0.0.0.3 area 0

ip nat inside source list 1 interface Serial0/1/0 overload
access-list 1 permit 192.168.100.0 0.0.0.255

! HR (VLAN 20) blocked from both server subnets
access-list 110 deny ip 192.168.100.64 0.0.0.31 host 10.10.10.10
access-list 110 deny ip 192.168.100.64 0.0.0.31 192.168.100.144 0.0.0.7
access-list 110 permit ip any any

! Sales (VLAN 30) limited to HTTP/HTTPS/ICMP toward both server subnets
access-list 120 permit icmp 192.168.100.0 0.0.0.63 host 10.10.10.10
access-list 120 permit tcp 192.168.100.0 0.0.0.63 host 10.10.10.10 eq www
access-list 120 permit tcp 192.168.100.0 0.0.0.63 host 10.10.10.10 eq 443
access-list 120 permit icmp 192.168.100.0 0.0.0.63 192.168.100.144 0.0.0.7
access-list 120 permit tcp 192.168.100.0 0.0.0.63 192.168.100.144 0.0.0.7 eq www
access-list 120 permit tcp 192.168.100.0 0.0.0.63 192.168.100.144 0.0.0.7 eq 443
access-list 120 deny ip 192.168.100.0 0.0.0.63 host 10.10.10.10
access-list 120 deny ip 192.168.100.0 0.0.0.63 192.168.100.144 0.0.0.7
access-list 120 permit ip any any

banner motd ^CUNAUTHORIZED ACCESS PROHIBITED^C

line vty 0 4
 transport input ssh
```

### SW-Core (below R1 — DHCP Snooping + Dynamic ARP Inspection)

```
ip dhcp snooping vlan 10,20,30,40,50
ip dhcp snooping
ip arp inspection vlan 10,20,30,40,50

interface FastEthernet0/1
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust
interface FastEthernet0/2
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust
interface FastEthernet0/3
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust
interface FastEthernet0/4
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust

! Untrusted access ports rate-limited against DHCP flooding
interface range FastEthernet0/5 - 24
 ip dhcp snooping limit rate 10
```

### SW-Left (VLAN 10/IT — PC0, VLAN 20/HR — PC1)

```
ip dhcp snooping vlan 10,20,30,40,50
ip dhcp snooping

interface FastEthernet0/1
 switchport mode trunk
 ip dhcp snooping trust
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 20
```

### SW-Bottom (VLAN 30/Sales — PC2, PC3)

```
ip dhcp snooping vlan 10,20,30,40,50
ip dhcp snooping

interface FastEthernet0/1
 switchport mode trunk
 ip dhcp snooping trust
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 30
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 30
```

### SW-Right (VLAN 40/Management — PC4, VLAN 50/Servers — Server1)

```
ip dhcp snooping vlan 10,20,30,40,50
ip dhcp snooping

interface FastEthernet0/1
 switchport mode trunk
 ip dhcp snooping trust
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 40
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 50
 switchport port-security
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 0001.6309.BC71
```

### R2 (Head Office — routing + NAT to Server0)

```
interface GigabitEthernet0/0/0.50
 encapsulation dot1Q 50
 ip address 10.10.10.1 255.255.255.0
 ip nat outside

interface Serial0/1/0
 ip address 172.16.1.2 255.255.255.252
 ip nat inside

router ospf 1
 network 172.16.1.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0

ip nat inside source list 1 interface Serial0/1/0 overload
access-list 1 permit 10.10.10.0 0.0.0.255

line vty 0 4
 login
```

### SW-Server (below R2 — Server0 access)

```
interface FastEthernet0/1
 switchport mode trunk
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 50
```

## Verification
- [x] OSPF adjacency up between R1 and R2 across the `172.16.1.0/30` serial link
- [x] Each Branch VLAN (10/20/30/40/50) leases addresses from its R1 DHCP pool
- [x] Branch LAN (`192.168.100.0/24`) NATs/overloads out through R1's Serial0/1/0 toward Head Office
- [x] HR (VLAN 20) denied access to both Server0 (`10.10.10.10`) and Server1 (`192.168.100.144/29`) — ACL 110
- [x] Sales (VLAN 30) reaches both servers only via HTTP/HTTPS/ICMP — ACL 120
- [x] Management (VLAN 40) has unrestricted access
- [x] DHCP Snooping + Dynamic ARP Inspection active on SW-Core, trunk ports trusted, access ports rate-limited
- [x] Port Security (sticky MAC) enforced on Server1's access port
- [x] R1 remote management restricted to SSH, authenticated via TACACS+ (local account as fallback)

## Notes
- R1's hostname is configured as `h1`.
- R1 uses `enable password` rather than `enable secret`; consider tightening this if the lab is reused for a security-focused module.
- R2 does not yet mirror R1's hardening — its VTY lines use plain `login` with no AAA/SSH restriction, and Telnet is not explicitly disabled. Worth revisiting if Head Office is meant to be equally secured.
- The TACACS+ server (`192.168.100.146`) sits inside the Server1 subnet (`192.168.100.144/29`); confirm whether it's the same device as Server1 or a separate host.

## Skills Demonstrated
Router-on-a-Stick across five VLANs, VLSM subnetting, per-VLAN DHCP on the
branch router, Serial WAN with OSPF, NAT Overload (PAT), extended ACLs for
selective VLAN-to-server filtering, AAA authentication via TACACS+ with
local fallback, SSH-only remote management, DHCP Snooping, Dynamic ARP
Inspection, and Port Security with sticky MAC learning.
