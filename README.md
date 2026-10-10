# Hospital-Network
A multi-floor hospital network built and configured in Cisco Packet Tracer. Each floor has its own router and switch, departments are separated into VLANs, a single DHCP server hands out addresses to every floor, and OSPF keeps the routers in sync.

## What's implemented

- Three floors, each with a switch and a router, with the three routers connected in a triangle
- Separate VLANs for each department
- Router-on-a-stick inter-VLAN routing using 802.1Q subinterfaces
- A central DHCP server in the Floor 1 server room, with `ip helper-address` relay on every router so devices on all floors get addresses automatically
- OSPF dynamic routing between the floor routers

## Topology

![Hospital network topology in Packet Tracer](evidence/topology.png)

| Floor | Departments (VLAN) |
|-------|--------------------|
| Floor 1 | Doctors (40), Clinical Staff (20), IT (50), Server Room (60) with DHCP and DNS servers |
| Floor 2 | Administration (10), Doctors (40), IT (50) |
| Floor 3 | Administration (10), Clinical Staff (20), Nursing Staff (30) |

Each floor switch connects to its floor router over a single trunk link. The routers connect to each other in a triangle, so every floor has two paths to every other floor.

The full IP plan (subnets, gateways, DHCP ranges, and router link addressing) is in [`docs/addressing-table.md`](docs/addressing-table.md).

## How it works

**Inter-VLAN routing.** Each router has one subinterface per VLAN on its floor, tagged with `encapsulation dot1Q <vlan-id>`. The subinterface address is the default gateway for that VLAN, so traffic between departments goes up to the router and back down.

**DHCP.** One server in the Floor 1 server room holds a pool for each subnet. Because the server is on a different subnet from the user VLANs, each router relays DHCP requests to it with `ip helper-address`. The server picks the pool based on which subnet the relayed request came from.

**OSPF.** Each router advertises its connected networks, so every floor can reach every other floor and the server room without static routes.

## Verifying it

Run these on the routers and switches:


show vlan brief
show interfaces trunk
show ip interface brief
show ip route
show ip ospf neighbor

On a PC (Desktop > Command Prompt):


ipconfig /all
ping <address of a PC on another floor>
tracert <address of a PC on another floor>


The first `tracert` hop should be the PC's own gateway, which confirms traffic is going through the router. `show ip ospf neighbor` should list neighbors in the FULL state, and `show ip route` on any router should show OSPF (`O`) routes for the other floors.

### Screenshots

**OSPF neighbors** (`show ip ospf neighbor` on a floor router)

![OSPF neighbors](evidence/ospf-neighbors.png)

**VLANs** (`show vlan brief` on a floor switch)

![VLANs](evidence/vlans.png)

**DHCP pools** (the server's Services > DHCP tab)

![DHCP pools](evidence/dhcp-pools.png)

## Lessons learned

- **Set the encapsulation before the IP address.** IOS rejects an IP on a subinterface until `encapsulation dot1Q <vlan-id>` is set.
- **Each floor needs its own subnets.** If two floors reuse the same subnet, routers prefer their own connected copy and DHCP replies can be delivered to the wrong floor.
- **DHCP clients only get addresses from the pool matching the subnet the request arrives from.** If every PC gets an address from the server room's pool, the PCs are probably in the same VLAN as the server, or the relay is missing.
- **OSPF neighbors can look healthy while routing is broken.** Neighbors form over the router-to-router links, but the LAN subnets still have to be advertised with `network` statements.

## Limitations and next steps

This is a working core, not a finished design:

- No ACLs yet, so every VLAN can currently reach every other VLAN. A real hospital would restrict this, for example keeping administration away from clinical systems.
- No port security, SSH, or other device hardening
- One DHCP server, so Floors 2 and 3 can't renew leases if Floor 1 is unreachable
- No tested failover (for example, shutting a router link and confirming OSPF reroutes)

Planned: ACL segmentation with a written access policy, device hardening, and a documented failover test.

## Repository layout



├── README.md
├── hospital topology project/        Packet Tracer file
└── evidence/        screenshots (topology, OSPF neighbors, VLANs, DHCP pools) and verification output


## Opening the lab

Open the `.pkt` file in `hospital topology project/` with Cisco Packet Tracer.
