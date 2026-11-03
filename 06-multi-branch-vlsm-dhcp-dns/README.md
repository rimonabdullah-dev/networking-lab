# Lab 06: Multi-Branch Company Network (VLSM, VLANs, DHCP Relay, DNS)

**Tool:** Cisco Packet Tracer
**Course:** Computer Networking (Southeast University)
**Type:** Course assignment, completed in a previous semester

## Objective

Design and configure a company network with a main office and two branch offices:

- Divide address space efficiently using **VLSM**.
- Separate departments in the main office with **VLANs** and connect them to the router with router-on-a-stick.
- Centralize **DHCP** and **DNS** on one server in the main office, and serve the branches through a **DHCP relay** (`ip helper-address`).
- Connect the branches to the main router and verify end-to-end connectivity.

## Files

| File | Description |
|---|---|
| [multi-branch-network.pkt](multi-branch-network.pkt) | Packet Tracer project file |
| [multi-branch-network.pdf](multi-branch-network.pdf) | Topology, configs, DHCP results, ping/tracert output and PDU simulation screenshots |

## Topology

`Main_R` (main router) connects to the main-office switch `Main_SW` through a trunk, and to two branch routers (`Br1_R`, `Br2_r`) over /30 links. Each branch has its own switch and PCs. The server (DHCP + DNS) sits in the main office. The full diagram is in [multi-branch-network.pdf](multi-branch-network.pdf).

## Addressing Plan (VLSM)

### Main office (192.168.0.0/24 split into four VLANs)

| VLAN | Name | Subnet | Mask | Gateway (Main_R sub-interface) |
|---|---|---|---|---|
| 20 | engineering | 192.168.0.0/25 | 255.255.255.128 | 192.168.0.1 (`Gig0/2.20`) |
| 10 | sales | 192.168.0.128/26 | 255.255.255.192 | 192.168.0.129 (`Gig0/2.10`) |
| 30 | HR | 192.168.0.192/27 | 255.255.255.224 | 192.168.0.193 (`Gig0/2.30`) |
| 40 | server | 192.168.0.224/28 | 255.255.255.240 | 192.168.0.225 (`Gig0/2.40`) |

### Branches and router links (192.168.1.0/24)

| Network | Purpose | Addresses |
|---|---|---|
| 192.168.1.0/27 | Branch 1 LAN | gateway 192.168.1.1 (`Br1_R Gig0/1`) |
| 192.168.1.32/27 | Branch 2 LAN | gateway 192.168.1.33 (`Br2_r Gig0/1`) |
| 192.168.1.64/30 | Main_R ↔ Br1_R | 192.168.1.65 (Main_R `Gig0/0`), 192.168.1.66 (Br1_R `Gig0/0`) |
| 192.168.1.68/30 | Main_R ↔ Br2_r | 192.168.1.69 (Main_R `Gig0/1`), 192.168.1.70 (Br2_r `Gig0/0`) |

## Configuration

### Main_SW (main-office switch)

```
interface range fastEthernet 0/2 - 3
 switchport mode access
 switchport access vlan 10
interface range fastEthernet 0/4 - 5
 switchport mode access
 switchport access vlan 20
interface range fastEthernet 0/6 - 7
 switchport mode access
 switchport access vlan 30
interface fastEthernet 0/8
 switchport mode access
 switchport access vlan 40
interface fastEthernet 0/1
 switchport mode trunk
```

| VLAN | Name | Ports |
|---|---|---|
| 10 | sales | Fa0/2, Fa0/3 |
| 20 | engineering | Fa0/4, Fa0/5 |
| 30 | HR | Fa0/6, Fa0/7 |
| 40 | server | Fa0/8 |

### Main_R (main router): `show ip interface brief`

| Interface | IP address |
|---|---|
| GigabitEthernet0/0 | 192.168.1.65 |
| GigabitEthernet0/1 | 192.168.1.69 |
| GigabitEthernet0/2.10 | 192.168.0.129 |
| GigabitEthernet0/2.20 | 192.168.0.1 |
| GigabitEthernet0/2.30 | 192.168.0.193 |
| GigabitEthernet0/2.40 | 192.168.0.225 |

`Gig0/2` has no IP of its own; one sub-interface per VLAN is used (router-on-a-stick).

### Br1_R (branch 1 router)

```
interface GigabitEthernet0/0
 ip address 192.168.1.66 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 192.168.1.1 255.255.255.224
 ip helper-address 192.168.0.226
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 192.168.1.65
```

### Br2_r (branch 2 router)

| Interface | IP address |
|---|---|
| GigabitEthernet0/0 | 192.168.1.70 |
| GigabitEthernet0/1 | 192.168.1.33 |

### Server (DHCP + DNS)

| Setting | Value |
|---|---|
| IP address | 192.168.0.226 |
| Subnet mask | 255.255.255.240 |
| Default gateway | 192.168.0.225 |
| DNS server | 192.168.0.226 |
| DNS A record | `www.company.com` → 192.168.0.226 |

DHCP pools on the server: `Br1`, `Br2`, `sales`, `eng`, `hr`, `serverPool`.

## DHCP Results

| Client | IP address | Mask | Gateway | DNS |
|---|---|---|---|---|
| Br1_Pc1 | 192.168.1.2 | 255.255.255.224 | 192.168.1.1 | 192.168.0.226 |
| Br2_Pc1 | 192.168.1.35 | 255.255.255.224 | 192.168.1.33 | 192.168.0.226 |
| Sales1 | 192.168.0.130 | 255.255.255.192 | 192.168.0.129 | 192.168.0.226 |
| Eng1 | 192.168.0.2 | 255.255.255.128 | 192.168.0.1 | 192.168.0.226 |
| HR1 | 192.168.0.194 | 255.255.255.224 | 192.168.0.193 | 192.168.0.226 |

Every client received its own subnet's address and gateway, plus the central DNS server, from the single DHCP server.

## Verification

From `Sales1`:

```
C:\> ping 192.168.0.226       -> 4/4 replies (server, TTL=127)
C:\> ping 192.168.0.2         -> 3/4 replies (Eng1, VLAN 20, TTL=127)
C:\> ping www.company.com     -> resolves to 192.168.0.226, 4/4 replies
C:\> tracert 192.168.1.35     -> 192.168.0.129 -> 192.168.1.70 -> 192.168.1.35
```

- The first ping to `192.168.0.2` times out because ARP is still resolving.
- `ping www.company.com` succeeding confirms DNS resolution works.
- The traceroute shows the path from the main office to Branch 2 through Main_R and Br2_r.

### Packet Tracer simulation (Sales1 → Eng2, different VLANs)

The PDU details in [multi-branch-network.pdf](multi-branch-network.pdf) show:

- The frame arrives at `Main_R` on `Gig0/2` with an **802.1Q (Dot1q) tag**.
- The router sends it back out the same `Gig0/2` port with a new tag and new MAC addresses, then the reply returns the same way.
- This is inter-VLAN routing through sub-interfaces.

## Key Learnings

- **VLSM** gives each subnet only the size it needs, by choosing a different mask per network (/25, /26, /27, /28, /30).
- Router-on-a-stick uses one trunk link and one sub-interface per VLAN.
- DHCP broadcasts do not cross routers. `ip helper-address <server>` forwards them to a DHCP server in another subnet.
- A **default route** (`ip route 0.0.0.0 0.0.0.0 <next-hop>`) sends everything unknown toward the main router.
- Hosts should get DNS from DHCP, so names like `www.company.com` work without manual setup.
- `tracert` shows each router hop, and the PDU simulation shows how the Layer 2 header changes at every router.
