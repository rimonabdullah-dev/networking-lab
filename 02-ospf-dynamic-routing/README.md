# Lab 02: Dynamic Routing with OSPF

**Tool:** Cisco Packet Tracer
**Course:** Computer Networking (Southeast University)
**Completed:** Previous semester

## Objective

Configure single-area OSPF (Area 0) on three routers so that all networks learn routes automatically and every PC can reach every other PC.

## Files

| File | Description |
|---|---|
| [ospf.pkt](ospf.pkt) | Packet Tracer project file |
| [ospf.pdf](ospf.pdf) | Topology, router configs, OSPF database and ping screenshots |

## Topology

3 routers (Router1, Router2, Router3), 3 switches and 3 PCs. Router2 and Router3 are connected by a serial link, the other router-to-router links are Gigabit Ethernet. The full diagram is in [ospf.pdf](ospf.pdf).

| Network | Connects |
|---|---|
| 192.168.0.0/24 | Router2 ↔ Router3 (Serial) |
| 192.168.1.0/24 | Router1 ↔ Router3 |
| 192.168.2.0/24 | Router1 ↔ Router2 |
| 192.168.3.0/30 | Router3 ↔ Switch2 ↔ PC3 |
| 192.168.4.0/30 | Router1 ↔ Switch3 ↔ PC2 |
| 192.168.5.0/30 | Router2 ↔ Switch0 ↔ PC1 |

## Configuration

### Router1 (hostname r1, router-id 1.1.1.1)

```
interface GigabitEthernet0/1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
interface GigabitEthernet0/2
 ip address 192.168.4.1 255.255.255.252
 no shutdown
!
hostname r1
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 192.168.2.0 0.0.0.255 area 0
 network 192.168.4.0 0.0.0.3 area 0
```

### Router3 (hostname r3, router-id 3.3.3.3)

```
interface GigabitEthernet0/1
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface GigabitEthernet0/2
 ip address 192.168.3.1 255.255.255.252
 no shutdown
!
hostname r3
router ospf 1
 router-id 3.3.3.3
 network 192.168.0.0 0.0.0.255 area 0
 network 192.168.1.0 0.0.0.255 area 0
 network 192.168.3.0 0.0.0.3 area 0
```

### Router2 (hostname r2, router-id 2.2.2.2)

Router2 uses the same method: `router ospf 1`, `router-id 2.2.2.2`, and a `network` statement (with wildcard mask, area 0) for each of its connected networks. Its details are in [ospf.pdf](ospf.pdf) and [ospf.pkt](ospf.pkt).

## Verification

```
PC2> ping 192.168.3.2     -> 4/4 replies (0% loss)
PC1> ping 192.168.3.2     -> 3/4 replies
```

The first ping can time out because ARP is still resolving. The remaining packets succeed, which confirms OSPF has distributed the routes.

On Router3, the console shows OSPF neighbors reaching the FULL state with Router1 (1.1.1.1) and Router2 (2.2.2.2).

On Router2, `show ip ospf database` lists three Router LSAs (1.1.1.1, 2.2.2.2, 3.3.3.3) and two Network LSAs in Area 0, so all three routers share the same link-state database.

Useful commands:

```
show ip route            (O = route learned via OSPF)
show ip ospf neighbor
show ip ospf database
show ip protocols
```

## Key Learnings

- OSPF is a link-state protocol: every router builds the same map (LSDB) and runs SPF on it.
- `network` statements use a **wildcard mask** (inverse of the subnet mask): /24 = `0.0.0.255`, /30 = `0.0.0.3`.
- A manually set `router-id` keeps router identity stable.
- Neighbors move through LOADING to FULL before routes are exchanged.
- OSPF chooses paths by cost (based on bandwidth), not hop count like RIP.
