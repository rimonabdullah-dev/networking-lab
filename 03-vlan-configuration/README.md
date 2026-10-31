# Lab 03: VLAN Configuration with Trunking

**Tool:** Cisco Packet Tracer
**Course:** Computer Networking (Southeast University)
**Completed:** Previous semester

## Objective

Create two VLANs on two switches, assign access ports to each VLAN, and connect the switches with a trunk link so that devices in the same VLAN can communicate across both switches.

## Files

| File | Description |
|---|---|
| [vLAN.pkz](vLAN.pkz) | Packet Tracer project file |
| [vLAN.pdf](vLAN.pdf) | Topology, switch configs, `show vlan brief` output and ping screenshots |

## Topology

Two 2960 switches (Switch0 on the left, Switch1 on the right) connected through their `Gig0/1` ports. The full diagram is in [vLAN.pdf](vLAN.pdf).

| VLAN | Name | Subnet | Access ports |
|---|---|---|---|
| 10 | students | 192.168.1.0/24 | Fa0/2, Fa0/3 |
| 20 | faculties | 192.168.2.0/24 | Fa0/11, Fa0/12 |

| Switch | Port | Device | IP address | VLAN |
|---|---|---|---|---|
| Switch0 | Fa0/2 | Laptop | 192.168.1.10/24 | 10 |
| Switch0 | Fa0/3 | Laptop | 192.168.1.11/24 | 10 |
| Switch0 | Fa0/11 | Laptop | 192.168.2.10/24 | 20 |
| Switch0 | Fa0/12 | Laptop | 192.168.2.11/24 | 20 |
| Switch1 | Fa0/2 | PC | 192.168.1.12/24 | 10 |
| Switch1 | Fa0/11 | PC | 192.168.2.12/24 | 20 |

## Configuration

### Switch0

```
vlan 10
 name students
vlan 20
 name faculties
exit
!
interface range fastEthernet 0/2 - 3
 switchport mode access
 switchport access vlan 10
interface range fastEthernet 0/11 - 12
 switchport mode access
 switchport access vlan 20
!
interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

### Switch1

```
vlan 10
 name students
vlan 20
 name faculties
exit
!
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 10
interface fastEthernet 0/11
 switchport mode access
 switchport access vlan 20
!
interface gigabitEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

VLANs must be created on **both** switches, and the trunk must be configured on **both** ends.

## Verification

```
show vlan brief
```

Output shows VLAN 10 (students) on Fa0/2, Fa0/3 and VLAN 20 (faculties) on Fa0/11, Fa0/12 of Switch0, and VLAN 10 on Fa0/2, VLAN 20 on Fa0/11 of Switch1.

```
C:\> ping 192.168.1.10    (from 192.168.1.12, Switch1 -> Switch0, same VLAN 10)
```

Result: 4/4 replies (0% loss), so VLAN 10 traffic crosses the trunk.

Useful commands:

```
show vlan brief
show interfaces trunk
```

## Troubleshooting Note

While configuring Switch1, the console showed:

```
%SPANTREE-2-RECV_PVID_ERR: Received 802.1Q BPDU on non trunk GigabitEthernet0/1 VLAN1
```

This appeared because Switch0's port was already a trunk while Switch1's port was still an access port. It cleared after `switchport mode trunk` was set on Switch1 as well.

## Key Learnings

- A VLAN splits one physical switch into separate broadcast domains.
- Access port = carries one VLAN (PC/laptop side). Trunk port = carries multiple VLANs (switch-to-switch link).
- `switchport trunk allowed vlan 10,20` limits which VLANs may cross the trunk.
- Both ends of a trunk must be set to trunk mode, or spanning tree reports an inconsistent port type.
- Different VLANs cannot talk to each other without a router (inter-VLAN routing).
