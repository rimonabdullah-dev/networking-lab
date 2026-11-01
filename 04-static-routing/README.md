# Lab 04: Static Routing with DHCP, DNS and Web Server

**Tool:** Cisco Packet Tracer
**Course:** Computer Networking (Southeast University)
**Completed:** Previous semester

## Objective

Connect three routers in a chain using static routes, and provide network services across the whole topology: DHCP for automatic addressing, a DNS server, and a web server.

## Files

| File | Description |
|---|---|
| [static-routing.pkt](static-routing.pkt) | Packet Tracer project file |
| [static-routing.pdf](static-routing.pdf) | Topology, router configs, DHCP client settings and ping screenshots |

## Topology

3 routers in a chain (Router0, Router1, Router2), 3 switches, laptops, a PC and three servers (DHCP, DNS, web). The full diagram is in [static-routing.pdf](static-routing.pdf).

| Network | Connects |
|---|---|
| 192.168.1.0/24 | Router0 ↔ Switch0 ↔ Laptop0, Laptop1 |
| 192.168.2.0/24 | Router0 ↔ Router1 |
| 192.168.3.0/24 | Router1 ↔ Router2 |
| 192.168.4.0/24 | Router2 ↔ Switch2 ↔ PC0, DNS server (192.168.4.2), DHCP server (192.168.4.3) |
| 192.168.5.0/24 | Router1 ↔ Switch1 ↔ Laptop2, web server |

## Configuration

### Router1 (middle router): DHCP and static routes

```
ip dhcp excluded-address 192.168.5.1 192.168.5.5
ip dhcp pool lan2
 network 192.168.5.0 255.255.255.0
 default-router 192.168.5.1
 dns-server 192.168.4.2
exit
!
ip route 192.168.4.0 255.255.255.0 192.168.3.2
ip route 192.168.1.0 255.255.255.0 192.168.2.1
```

### Router2 (right router): static routes

```
ip route 192.168.5.0 255.255.255.0 192.168.3.1
ip route 192.168.2.0 255.255.255.0 192.168.3.1
ip route 192.168.1.0 255.255.255.0 192.168.3.1
```

### Router0 (left router)

Its configuration is in [static-routing.pdf](static-routing.pdf) and [static-routing.pkt](static-routing.pkt).

## DHCP Results

| Client | IP address | Gateway | DNS server |
|---|---|---|---|
| PC0 | 192.168.4.10 | 192.168.4.1 | 192.168.4.2 |
| Laptop0 | 192.168.1.6 | 192.168.1.1 | 192.168.4.2 |

## Verification

```
C:\> ping 192.168.4.10    -> 4/4 replies (0% loss), TTL=125
C:\> ping 192.168.5.6     -> 4/4 replies (0% loss), TTL=126
```

Replies from other networks confirm the static routes work in both directions. TTL drops by one at every router crossed, which matches the number of routers between the hosts.

Useful commands:

```
show ip route            (S = static route)
show ip dhcp binding
ipconfig /all
```

## Troubleshooting Note

A route to `192.168.1.0` was first entered with next hop `192.168.1.1`, which is the destination network's own gateway. It was removed with `no ip route ...` and replaced with next hop `192.168.2.1`.

Rule: the next hop must be the IP of the **neighboring router on the shared link**, not an address inside the destination network.

## Key Learnings

- Static routes are written by hand: `ip route <network> <mask> <next-hop>`.
- Every router needs a route to every network it is not directly connected to, otherwise replies cannot return.
- A router can act as a DHCP server: `ip dhcp excluded-address`, then `ip dhcp pool` with `network`, `default-router` and `dns-server`.
- Reserve addresses (excluded-address) for gateways and servers so DHCP does not hand them out.
- DHCP gives clients the DNS server address too, so names can be resolved without manual setup.
