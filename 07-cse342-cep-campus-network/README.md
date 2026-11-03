# CSE342 Complex Engineering Project: Two-Building Campus Network

**Tool:** Cisco Packet Tracer
**Course:** CSE342 Computer Networking Lab, Southeast University
**Semester:** Summer 2026

## Objective

Design, address and configure a network for an institute with two buildings and an outside (ISP) network:

- Divide `192.168.0.0/16` into subnets with **VLSM**, one per unit.
- Separate units with **VLANs** and trunks, and route between them with **router-on-a-stick**.
- Connect the two buildings with **OSPF** and serve clients with **DHCP**.
- Provide server-room services: **DNS, HTTP, FTP and mail (SMTP/POP3)**.
- Secure and expose the network with **PAT, static NAT and an ACL**.

## Files

| File | Description |
|---|---|
| [campus-network.pkt](campus-network.pkt) | Packet Tracer project file |
| [cse342-cep-report.pdf](cse342-cep-report.pdf) | Full report with topology, configs, verification and troubleshooting record |

## Topology

Two buildings connected through routers **R1** and **R2**. R1 also connects to an **ISP** router that represents the internet, with `PC12` on the ISP side. R1 serves the main building through a trunk and sub-interfaces; R2 serves the Innovation Lab building. The diagram is in [cse342-cep-report.pdf](cse342-cep-report.pdf).

## Addressing Plan (VLSM)

| Unit | VLAN | Network | Mask | Usable hosts | Gateway |
|---|---|---|---|---|---|
| Admin | 10 | 192.168.10.0 | /27 | .1 – .30 | 192.168.10.1 |
| Accounts | 20 | 192.168.20.0 | /28 | .1 – .14 | 192.168.20.1 |
| Faculty | 30 | 192.168.30.0 | /27 | .1 – .30 | 192.168.30.1 |
| Student Lab | 40 | 192.168.40.0 | /25 | .1 – .126 | 192.168.40.1 |
| Server Room | 50 | 192.168.50.0 | /28 | .1 – .14 | 192.168.50.1 |
| Innovation Lab | 60 | 192.168.60.0 | /27 | .1 – .30 | 192.168.60.1 (on R2) |
| R1 ↔ R2 link | - | 192.168.100.0 | /30 | .1 – .2 | - |
| R1 ↔ ISP link | - | 203.0.113.0 | /29 | .1 – .6 | - |

Student Lab needs about 100 hosts, so `/25` is used. Router links use `/30`. Note from the report: the Faculty subnet has no spare room and could be widened to `/26` later.

## Switch Configuration

| Switch | VLANs | Access ports | Trunk |
|---|---|---|---|
| Switch0 (SW1) | 10 `admin`, 20 `accounts` | Fa0/3-4 → VLAN 10, Fa0/5-6 → VLAN 20 | Fa0/1, Fa0/2 |
| Switch1 (SW2) | 30 `faculty`, 40 `student_lab`, 50 `server_room` | Fa0/2-3 → 30, Fa0/4-6 → 40, Fa0/7-10 → 50 | Fa0/1 |
| Switch2 (SW3) | 60 `innovation_lab` | Fa0/1-4 → VLAN 60 | none |

Example (Switch0):

```
vlan 10
 name admin
vlan 20
 name accounts
interface range fastEthernet 0/3 - 4
 switchport mode access
 switchport access vlan 10
interface range fastEthernet 0/5 - 6
 switchport mode access
 switchport access vlan 20
interface fastEthernet 0/1
 switchport mode trunk
interface fastEthernet 0/2
 switchport mode trunk
```

`show interfaces trunk` confirms the trunks are in 802.1q mode and trunking.

## Router Configuration

### R1: router-on-a-stick, default route

```
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.248
interface GigabitEthernet0/1
 ip address 192.168.100.1 255.255.255.252
!
interface GigabitEthernet0/2.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.224
 ip nat inside
! (same pattern for .20, .30 and .40 with /28, /27 and /25)
interface GigabitEthernet0/2.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.240
 ip nat inside
!
ip route 0.0.0.0 0.0.0.0 203.0.113.2
```

### R2

```
interface GigabitEthernet0/0
 ip address 192.168.100.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 192.168.60.1 255.255.255.224
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 192.168.100.1
```

## OSPF

R1:

```
router ospf 1
 network 192.168.10.0 0.0.0.31 area 0
 network 192.168.20.0 0.0.0.15 area 0
 network 192.168.30.0 0.0.0.31 area 0
 network 192.168.40.0 0.0.0.127 area 0
 network 192.168.50.0 0.0.0.15 area 0
 network 192.168.100.0 0.0.0.3 area 0
```

R2:

```
router ospf 1
 network 192.168.60.0 0.0.0.31 area 0
 network 192.168.100.0 0.0.0.3 area 0
```

Verification on R2: the OSPF neighbor (R1) is in `FULL/DR` state, and every VLAN subnet of the main building appears as an `O` route via `192.168.100.1`.

## DHCP (on R1, Student Lab)

```
ip dhcp excluded-address 192.168.40.1 192.168.40.10
ip dhcp pool student_lab
 network 192.168.40.0 255.255.255.128
 default-router 192.168.40.1
 dns-server 192.168.50.2
```

## Server Room Services

| Server | Address | Service |
|---|---|---|
| DNS server | 192.168.50.2 | DNS and HTTP |
| Web server | 192.168.50.3 | HTTP/HTTPS, shows the "NETWORKING ASSIGNMENT" page |
| FTP server | in 192.168.50.0/28 | FTP with two user accounts (credentials omitted here) |
| Mail server | 192.168.50.5 | SMTP + POP3, domain `networking.com`, users `admin`, `admin1`, `faculty1`, `student1` |

Mail test: a mail was sent between the Admin and Faculty users and received in the inbox.

## NAT and ACL (on R1)

```
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface gigabitEthernet 0/0 overload
ip nat inside source static 192.168.50.3 203.0.113.3
!
access-list 101 permit tcp any any eq 80
access-list 101 permit tcp any any eq 22
access-list 101 permit tcp any any eq 53
access-list 101 deny ip any any
!
interface gigabitEthernet 0/2.10
 ip access-group 101 in
```

| Feature | Purpose |
|---|---|
| PAT (overload) | All internal `192.168.0.0/16` hosts share the R1 outside address |
| Static NAT | Maps the web server `192.168.50.3` to the public address `203.0.113.3` |
| ACL 101 (Admin VLAN, inbound) | Allows only HTTP (80), SSH (22) and port 53 over TCP; denies everything else |

## Verification

| Test | Result |
|---|---|
| `ping 192.168.50.3` from ADMIN-PC1 | 100% loss, "Destination host unreachable" from 192.168.10.1: ICMP is blocked by ACL 101 |
| `http://192.168.50.3` from ADMIN-PC1 | Web page opens: HTTP is allowed by ACL 101 |
| `http://203.0.113.3` from PC12 (ISP side) | Web page opens: the static NAT works from outside |
| `show ip nat translations` on R1 | Static entry `203.0.113.3` ↔ `192.168.50.3` |
| `show access-lists` on R1 | ACL 101 shows 6 matches on `www` and 4 matches on `deny ip any any` |
| `show ip route` on R2 | All main-building VLAN subnets learned as `O` routes |

## Troubleshooting Record

| Problem | Root cause | Fix |
|---|---|---|
| Router rejected a secondary IP on `Gig0/0` | Primary and secondary address were in the same subnet | Removed the extra secondary address line |
| "Overlaps" error on `Gig0/2.10` | The VLAN 20 address was typed on the VLAN 10 sub-interface | Removed the wrong address and applied `192.168.10.1` |
| Student Lab PCs got no IP from DHCP | VLAN 40 was never created on SW1, so the trunk did not forward VLAN 40 traffic to R1 | Created VLANs 30, 40 and 50 on SW1 (no port assignment needed); the PCs then received addresses |

## Suggested Improvement

R1 is the only path between the two buildings and the ISP. A second redundant link between R1 and R2, plus basic network monitoring, would make the network more reliable if a link or device fails.

## Key Learnings

- A trunk only forwards VLANs that exist on **every switch along the path**, even if the switch has no access ports in that VLAN.
- OSPF `network` statements use wildcard masks: `/27` = `0.0.0.31`, `/28` = `0.0.0.15`, `/25` = `0.0.0.127`, `/30` = `0.0.0.3`.
- **PAT** shares one public address among many hosts; **static NAT** exposes one inside server to the outside.
- An **ACL** is processed top to bottom and ends with an implicit deny, so the explicit `deny ip any any` also lets you count the blocked packets.
- Apply an inbound ACL on the interface closest to the source it should filter.
- Keep a troubleshooting record: problem, investigation, root cause, correction, verification.
