# Lab 01: Dynamic Routing with RIP v2

**Tool:** Cisco Packet Tracer
**Course:** Computer Networking (Southeast University)
**Completed:** Previous semester

## Objective

Configure RIP v2 on three routers so that every network can reach every other network without manually writing static routes.

## Files

| File | Description |
|---|---|
| [Dynamic_Routing.pkt](Dynamic_Routing.pkt) | Packet Tracer project file |
| [Dynamic_Routing.pdf](Dynamic_Routing.pdf) | Topology, router configs and ping screenshots |

## Topology

3 routers (Router0, Router1, Router2), 4 switches, 2 PCs and 2 laptops. Router0 and Router1 are connected by a serial link, the other links are Gigabit Ethernet. The full diagram is in [Dynamic_Routing.pdf](Dynamic_Routing.pdf).

| Network | Connects |
|---|---|
| 192.168.0.0/24 | Router1 ↔ Switch1 ↔ Laptop1 |
| 192.168.1.0/24 | Router0 ↔ Switch0 ↔ PC0 |
| 192.168.2.0/24 | Router0 ↔ Switch3 ↔ PC1 |
| 192.168.3.0/24 | Router2 ↔ Switch2 ↔ Laptop0 |
| 192.168.4.0/30 | Router2 ↔ Router1 |
| 192.168.5.0/30 | Router0 ↔ Router1 (Serial) |
| 192.168.6.0/30 | Router0 ↔ Router2 |

## Configuration

### Router0

```
interface GigabitEthernet0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface GigabitEthernet0/2
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface Serial0/3/0
 ip address 192.168.5.1 255.255.255.252
 no shutdown
!
router rip
 version 2
 network 192.168.1.0
 network 192.168.2.0
 network 192.168.5.0
 network 192.168.6.0
```

### Router1

```
interface GigabitEthernet0/1
 ip address 192.168.0.1 255.255.255.0
 no shutdown
interface Serial0/3/0
 ip address 192.168.5.2 255.255.255.252
 no shutdown
!
router rip
 version 2
 network 192.168.0.0
 network 192.168.4.0
 network 192.168.5.0
```

### Router2

```
interface GigabitEthernet0/0
 ip address 192.168.6.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 192.168.4.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 ip address 192.168.3.1 255.255.255.252
 no shutdown
!
router rip
 version 2
 network 192.168.3.0
 network 192.168.4.0
 network 192.168.6.0
```

## Verification

```
C:\> ping 192.168.1.2     -> 4/4 replies (0% loss)
C:\> ping 192.168.0.2     -> PC0 to Laptop1: 3/4 replies
```

The first ping times out because the router is still resolving ARP. The remaining packets succeed, which confirms RIP has distributed the routes.

Useful commands:

```
show ip route        (R = route learned via RIP)
show ip protocols
```

## Key Learnings

- RIP v2 is classless, so `version 2` must be set explicitly.
- The `network` command only lists directly connected networks.
- Routers learn remote networks automatically, so no manual `ip route` is needed.
- RIP uses hop count as its metric (maximum 15 hops).
