# Design Decisions

**Updated:** 2026-11-21

This document explains the main architectural decisions behind the current GNS3 Network Security Homelab.

---

## 1. Three-Router OSPF Core

### Decision

R1, R2, and R3 form a redundant OSPF Area 0 triangle.

The core transit networks are:

| Link | Network |
|---|---|
| R1-R2 | `10.10.100.0/30` |
| R2-R3 | `10.10.100.4/30` |
| R1-R3 | `10.10.100.8/30` |

### Why

The lab originally used static routing between routers. OSPF was introduced to make the routed core dynamic and to practice:

- Neighbor adjacency formation
- Route advertisement
- Path selection
- Link-state routing
- Redundant paths
- Reconvergence after a link failure

The direct R1-R3 link provides an alternate path if one of the paths through R2 becomes unavailable.

### Trade-off

OSPF is more complex than static routing and requires additional troubleshooting knowledge, but it provides much better scalability and resilience.

---

## 2. /30 Networks for Point-to-Point Transit Links

### Decision

Router-to-router and router-to-firewall transit links use dedicated `/30` networks.

Examples:

| Transit | Network |
|---|---|
| R1-R2 | `10.10.100.0/30` |
| R2-R3 | `10.10.100.4/30` |
| R1-R3 | `10.10.100.8/30` |
| R1-pfSense-Internal | `10.10.252.0/30` |
| R3-pfSense-Guest | `10.10.253.0/30` |
| R2-FortiGate | `10.10.254.0/30` |

### Why

Each of these links only needs two usable IPv4 addresses.

Using separate point-to-point subnets:

- Clearly identifies each Layer 3 link
- Prevents unrelated interfaces from sharing the same broadcast domain
- Makes routing tables easier to understand
- Makes failures easier to isolate
- Mirrors common enterprise/WAN design practices

A `/30` provides exactly two usable host addresses, which fits the current lab design.

---

## 3. Moving Employee Default Gateways from R1 to pfSense-Internal

### Decision

The original router-on-a-stick design on R1 was retired.

pfSense-Internal now owns the employee gateway addresses:

| Network | Gateway |
|---|---|
| IT | `10.10.10.1` |
| Finance | `10.10.40.1` |
| HR | `10.10.50.1` |

The old R1 VLAN subinterfaces were removed.

### Why

When R1 acted as the default gateway for all employee VLANs, inter-VLAN routing happened directly on the router.

That worked for learning VLAN routing, but it made security segmentation less realistic because traffic could be routed between departments without passing through a dedicated stateful firewall.

Moving the gateway addresses to pfSense-Internal means traffic between protected VLANs now crosses the firewall policy engine.

This allows the lab to enforce policies such as:

- IT can reach Finance
- IT can reach HR
- Finance cannot initiate connections to IT or HR
- HR cannot initiate connections to IT or Finance

### Trade-off

The design introduces more routing complexity because R1 is no longer directly connected to the employee networks.

That requires static routes between R1 and pfSense-Internal.

---

## 4. Keeping pfSense Outside the OSPF Core

### Decision

pfSense-Internal and pfSense-Guest do not participate directly in OSPF.

Instead:

- R1 uses static routes to employee networks through pfSense-Internal.
- R3 uses a static route to the Guest network through pfSense-Guest.
- R1 and R3 selectively redistribute those static routes into OSPF.

### Why

This keeps routing responsibilities separated:

- VyOS routers handle the dynamic OSPF core.
- pfSense handles gateway and security-policy enforcement.

It also provides hands-on practice with route redistribution without making every device participate in the same routing protocol.

Current R1 static routes:

| **Destination network** | **Next hop** |
|---|---|
| `10.10.10.0/24` | via `10.10.252.2` |
| `10.10.40.0/24` | via `10.10.252.2` |
| `10.10.50.0/24` | via `10.10.252.2` |

Current R3 static route:

| **Destination network** | **Next hop** |
|---|---|
| `10.10.20.0/24` | via `10.10.253.2` |

### Why Selective Redistribution

Only the intended protected networks should enter OSPF.

R1 therefore uses a prefix list and route map to redistribute the employee routes, while R3 selectively redistributes the Guest route.

This is safer and clearer than redistributing every static route automatically.

### Trade-off

The architecture requires more explicit route management than simply running OSPF everywhere.

However, that added complexity is useful for learning and gives tighter control over which routes enter the dynamic routing domain.

---

## 5. Keeping a Dedicated Server Network on R2

### Decision

DHCP01/DNS01 remains on a dedicated infrastructure network:

```text
Network: 10.10.30.0/24
Gateway: 10.10.30.1
Server:  10.10.30.10
```

R2 owns the server-network gateway.

### Why

The DHCP/DNS server is infrastructure, not an employee endpoint.

Keeping it on a dedicated network:

- Separates infrastructure services from user VLANs
- Gives all client networks a common service location
- Avoids duplicating DHCP and DNS servers in every VLAN
- Makes firewall exceptions for required services explicit

R2 advertises this server network into OSPF using its server-facing interface as a passive interface.

---

## 6. Centralized DHCP and DNS

### Decision

A single Ubuntu server, DHCP01, provides:

- Kea DHCPv4
- BIND9 DNS
- Internal `gns3.lab` DNS
- Recursive/forwarded public DNS resolution

### Why

A centralized service model more closely resembles an enterprise network than configuring independent DHCP/DNS services on every router or firewall.

It also creates useful routing and security requirements:

- Client broadcasts must be relayed across subnets.
- Firewalls must explicitly allow DNS traffic to `10.10.30.10`.
- All departments depend on shared infrastructure.

### Trade-off

The server becomes an important dependency. If DHCP01 or its network path fails, multiple client networks can be affected.

That is useful for future troubleshooting and resiliency exercises.

---

## 7. Running DHCP Relay on the Default-Gateway Firewalls

### Decision

DHCP relay was moved from the VyOS routers to the pfSense firewalls.

Current relay ownership:

| Network | Relay |
|---|---|
| IT | pfSense-Internal |
| Finance | pfSense-Internal |
| HR | pfSense-Internal |
| Guest | pfSense-Guest |

All relays forward to:

```text
10.10.30.10
```

### Why

The pfSense firewalls now own the client gateway interfaces, so they are the logical Layer 3 devices to receive DHCP broadcasts from those networks.

This keeps the network roles consistent:

- Clients send local broadcasts.
- Their local gateway/firewall receives them.
- The firewall relays them to the centralized DHCP server.

It also removes obsolete DHCP-relay configuration from R1 and R3.

---

## 8. Using FortiGate as the Perimeter Firewall

### Decision

FortiGate sits between R2 and the GNS3 NAT/Internet edge.

The internal transit is:

```text
FortiGate: 10.10.254.1/30
R2:        10.10.254.2/30
```

R2 uses FortiGate as its upstream default gateway.

### Why

This creates a layered firewall architecture:

```text
Internet
   |
FortiGate          <- perimeter security
   |
OSPF routed core
   |
pfSense            <- internal segmentation
   |
User networks
```

FortiGate and pfSense therefore have different responsibilities rather than duplicating the same role.

This also gives the lab experience with more than one firewall platform.

---

## 9. Originate the Internet Default Route from R2

### Decision

R2 has a static default route toward FortiGate:

| **Destination network** | **Next hop** |
|---|---|
| `0.0.0.0/0` | via `10.10.254.1` |

R2 then originates the default route into OSPF.

### Why

R2 is the core router directly connected to the perimeter firewall.

Rather than configuring separate Internet default routes manually on R1 and R3, R2 tells the OSPF domain that it is the path toward unknown/Internet destinations.

This keeps Internet routing centralized and allows R1/R3 to learn the default route dynamically.

---

## 10. Using Default-Deny Internal Firewall Policy

### Decision

pfSense rules are designed around default deny.

Specific required traffic is explicitly permitted before broader private-network blocks.

Examples include:

- DHCP
- DNS to `10.10.30.10`
- IT administrative access
- Internet access

### Why

A default-deny model is safer and easier to reason about than allowing broad communication and trying to block individual unwanted flows afterward.

The policy follows the pattern:

```text
required exception
        |
required exception
        |
block internal/private destinations
        |
allow Internet/non-private destinations
```

Rule order therefore matters.

---

## 11. Giving IT Broader Administrative Access

### Decision

The IT network receives broader access than Finance, HR, and Guest.

Current examples include:

- IT -> Finance
- IT -> HR
- IT -> Guest
- IT -> pfSense-Internal HTTPS management
- HQ IT -> pfSense-Guest HTTPS management

### Why

The IT department is treated as the administrative/support group responsible for troubleshooting and managing systems across the environment.

Finance and HR are treated as normal user departments and do not receive equivalent administrative access.

This provides a foundation for future remote-management exercises using technologies such as:

- SSH
- RDP
- HTTPS firewall administration
- PowerShell/WinRM

### Security Consideration

IT access should still be explicitly defined rather than permanently using an unrestricted IT-to-any policy.

As the lab grows, management traffic can be narrowed by protocol, destination aliases, or dedicated administrative hosts.

---

## 12. Managing pfSense-Internal from the IT VLAN

### Decision

The old pfSense-Internal management LAN:

```text
192.168.1.0/24
```

was retired.

The old `192.168.1.1/24` interface was removed.

pfSense-Internal is now managed through:

```text
https://10.10.10.1
Source: 10.10.10.0/24
TCP 443
```

### Why

The separate management LAN had become unnecessary after the IT VLAN was established as the administrative network.

Removing the unused interface:

- Simplifies the topology
- Removes an unused network
- Reduces unnecessary management exposure
- Keeps administrative access aligned with the IT security policy

---

## 13. Making webterm-IT a Normal DHCP IT Workstation

### Decision

`webterm-IT` no longer uses a dedicated static `192.168.1.x` management address.

It is now a normal DHCP client on VLAN 10.

### Why

The Webterm is intended to represent an IT employee/admin workstation rather than a special out-of-band firewall console.

Placing it in the same VLAN and DHCP environment as other IT clients makes testing more realistic.

It can now be used for:

- Browser-based firewall management
- DNS testing
- Internet testing
- Application-layer troubleshooting
- Future SSH/RDP administrative workflows

Infrastructure devices still use fixed addresses where predictability is required.

---

## 14. Keeping Guest Behind a Separate Firewall

### Decision

The Guest network uses a dedicated pfSense-Guest firewall rather than sharing pfSense-Internal.

Current Guest network:

```text
10.10.20.0/24
Gateway: 10.10.20.1
```

### Why

Guest devices should be treated as less trusted than employee systems.

A separate firewall provides a clear boundary and allows the lab to practice:

- Guest isolation
- Private-network blocking
- Internet-only access
- Explicit IT support exceptions
- Remote firewall management from a trusted IT network

### Current Administrative Exception

HQ IT (`10.10.10.0/24`) is explicitly permitted to manage pfSense-Guest using HTTPS to its WAN/transit address:

```text
https://10.10.253.2
TCP 443
```

Because this traffic crosses both segmentation firewalls, the management path is explicitly allowed at both enforcement points:

```text
pfSense-Internal:
10.10.10.0/24 -> 10.10.253.2 TCP/443 -> Allow

pfSense-Guest WAN:
10.10.10.0/24 -> WAN address TCP/443 -> Allow
```

This has been verified from the HQ IT network. IT is also allowed to reach Guest hosts for troubleshooting.

---

## 15. Leaving pfSense-Internal WAN Default-Deny as is

### Decision

No broad pass rules are configured on the pfSense-Internal WAN interface.

### Why

The WAN interface faces R1 and the routed core.

Allowed sessions initiated from IT, Finance, or HR create state entries, so legitimate reply traffic can return automatically.

Unsolicited traffic entering from the core has no matching WAN pass rule and is denied by default.

This provides a useful security boundary without adding redundant explicit block rules.

### Future Expansion

If a second branch is implemented, narrowly scoped WAN management rules can be added for trusted remote IT networks rather than opening the interface broadly.

---

## 16. Preserving Historical Designs Instead of Deleting Them

### Decision

Older topology diagrams and project logs are retained even when the design changes.

Examples include:

- Static-routing topology
- Earlier router-on-a-stick design
- Earlier DHCP/DNS milestones

### Why

The repository is intended to demonstrate progression and troubleshooting, not only the finished state.

Keeping historical artifacts shows:

- How the design evolved
- Why older configurations were replaced
- What concepts were learned at each stage
- The ability to migrate a functioning network instead of building only a final topology

Current-state documents are updated, while dated project logs remain historical records.

---

## 17. Restricting Perimeter Egress by Service

### Decision

FortiGate does not use a broad unrestricted internal-to-Internet allow rule.

The active `Internal-to-Internet` policy permits:

```text
HTTPS
DNS
NTP
PING
```

from the `INTERNAL-NETWORKS` object (`10.10.0.0/16`) toward the Internet-facing interface.

Traffic that does not match an explicit allow policy falls through to FortiGate's implicit deny behavior.

### Why

This creates a more deliberate perimeter-security model than simply allowing all outbound traffic.

It allows the lab to practice:

- service-based egress control
- default-deny policy behavior
- firewall rule matching
- NAT at the perimeter
- troubleshooting applications that fail because their required service is not allowed

### Trade-off

Some applications may fail even when basic Internet connectivity works because they use ports or protocols that are not currently permitted.

That behavior is intentional and should be evaluated before adding new services.

---

## 18. Centralizing Firewall Administration in the IT VLAN

### Decision

The IT VLAN is the trusted administrative network for browser-based firewall management.

Verified HTTPS management paths are:

```text
IT -> pfSense-Internal  10.10.10.1:443
IT -> pfSense-Guest     10.10.253.2:443
IT -> FortiGate         10.10.254.1:443
```

`webterm-IT` represents a normal DHCP-based IT administrative workstation rather than a dedicated out-of-band management host.

### Why

This models the operational role of an IT/network team more realistically:

- administrators work from a trusted user/admin network
- firewall GUIs are not broadly exposed to untrusted departments
- remote firewall management requires explicit policy exceptions
- the same workstation can troubleshoot protected endpoints and infrastructure

### Security Impact

Management access is intentionally restricted to the trusted IT source network and the required management destination/port.

For pfSense-Guest and FortiGate, the management path crosses routed/security boundaries, so the necessary upstream firewall rules must be present before the destination device can be reached.

---

## 19. Planned Multi-Site Expansion

### Status

**Planned — not yet implemented.**

### Direction

The current R3/Guest side may later be redesigned as a second business branch with its own:

- IT
- Finance
- HR

The intended future administrative model is for the IT teams at each site to be able to manage:

- The remote branch firewall
- Linux endpoints through SSH
- Windows endpoints through RDP
- Other infrastructure using explicitly permitted management protocols

### Design Principle

Cross-site management should use narrowly scoped firewall rules based on:

- Trusted IT source networks
- Specific destination networks or hosts
- Required management protocols and ports

The current Guest topology and addressing remain authoritative until that branch conversion is actually implemented and tested.

---

## Summary

The current architecture deliberately separates major responsibilities:

| Layer / Device | Primary Responsibility |
|---|---|
| FortiGate | Internet/perimeter security |
| R1/R2/R3 | Dynamic OSPF core routing |
| pfSense-Internal | Employee gateway and segmentation policy |
| pfSense-Guest | Guest gateway and isolation |
| DHCP01 | Centralized DHCP and DNS |
| Switch1/Switch3 | Employee VLAN transport |
| Switch2 | Guest access |
| IT VLAN | Trusted administrative/support network |

The design is intentionally more complex than the minimum required to provide connectivity. That complexity is used to practice realistic routing, segmentation, security policy, centralized services, controlled administration, migration, and troubleshooting.
