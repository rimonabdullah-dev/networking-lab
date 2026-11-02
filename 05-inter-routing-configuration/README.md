# Lab 05: Inter-VLAN Routing (Router-on-a-Stick)

**Tool:** Cisco Packet Tracer
**Course:** Computer Networking (Southeast University)
**Completed:** Previous semester

## Objective

Create two VLANs across two switches and use a single router interface with sub-interfaces (router-on-a-stick) so that devices in different VLANs can communicate with each other.

## Files

| File | Description |
|---|---|
| [inter-vlan-routing.pkt](inter-vlan-routing.pkt) | Packet Tracer project file |
| [inter-vlan-routing.pdf](inter-vlan-routing.pdf) | Topology, switch and router configs, and ping screenshots |

## Topology

One 2911 router (`Gig0/0`) connected to the left switch through a trunk port (`Fa0/6`). The two switches are linked by a trunk (`Fa0/5` on the left switch). The full diagram is in [inter-vlan-routing.pdf](inter-vlan-routing.pdf).

| VLAN | Name | Subnet | Gateway (router sub-interface) |
|---|---|---|---|
| 2 | home | 192.168.1.0/24 | 192.168.1.1 (`Gig0/0.2`) |
| 3 | office | 192.168.2.0/24 | 192.168.2.1 (`Gig0/0.3`) |

| Device | IP address | VLAN |
|---|---|---|
| Laptop (left switch, Fa0/1) | 192.168.1.2/24 | 2 |
| Laptop (left switch, Fa0/2) | 192.168.1.3/24 | 2 |
| Laptop (left switch, Fa0/3) | 192.168.2.2/24 | 3 |
| Laptop (left switch, Fa0/4) | 192.168.2.3/24 | 3 |
| PC (right switch) | 192.168.1.4/24 | 2 |
| PC (right switch) | 192.168.2.4/24 | 3 |

## Configuration

### Switch (left)

```
vlan 2
 name home
vlan 3
 name office
exit
!
interface range fastEthernet 0/1 - 2
 switchport mode access
 switchport access vlan 2
interface range fastEthernet 0/3 - 4
 switchport mode access
 switchport access vlan 3
!
interface fastEthernet 0/5
 switchport mode trunk
 switchport trunk allowed vlan 2,3
interface fastEthernet 0/6
 switchport mode trunk
 switchport trunk allowed vlan 2,3
```

### Router

```
interface gigabitEthernet 0/0
 no shutdown
!
interface gigabitEthernet 0/0.2
 encapsulation dot1Q 2
 ip address 192.168.1.1 255.255.255.0
!
interface gigabitEthernet 0/0.3
 encapsulation dot1Q 3
 ip address 192.168.2.1 255.255.255.0
```

Each device uses its own VLAN's sub-interface IP as its default gateway.

## Verification

```
C:\> ping 192.168.1.3     (from 192.168.2.3, VLAN 3 -> VLAN 2)  3/4 replies, TTL=127
C:\> ping 192.168.2.4     (from 192.168.1.2, VLAN 2 -> VLAN 3)  3/4 replies, TTL=127
```

The first ping times out because ARP is still resolving. The remaining packets succeed, so traffic is being routed between the VLANs. TTL=127 shows the packet crossed exactly one router.

Useful commands:

```
show vlan brief
show interfaces trunk
show ip interface brief
```

## Troubleshooting Notes

- `switchport trunk vlan 2,3` was rejected as invalid. The correct command is `switchport trunk allowed vlan 2,3`.
- On `Fa0/6` the allowed-VLAN line was also mistyped (`allowed trunk 2,3`). A trunk allows all VLANs by default, so traffic still flowed, but the correct command is `switchport trunk allowed vlan 2,3`.
- `interface fa0/0.2` failed because the router port is Gigabit. The correct interface is `gigabitEthernet 0/0.2`.

## Key Learnings

- Devices in different VLANs cannot communicate without a Layer 3 device.
- Router-on-a-stick: one physical router port is split into sub-interfaces, one per VLAN.
- `encapsulation dot1Q <vlan-id>` ties each sub-interface to a VLAN.
- The switch port facing the router must be a **trunk**.
- Each host's default gateway is the sub-interface IP of its own VLAN.
