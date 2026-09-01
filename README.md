# GNS3 Network Security Homelab

A mixed-vendor network engineering and cybersecurity homelab built using GNS3, Hyper-V, VyOS, Ubuntu Server, Kea DHCP, BIND9 DNS, VPCS, and GNS3 NAT.

## Project Goals

This project progressively builds an enterprise-style virtual network covering:

- IPv4 addressing and subnetting
- Static routing
- OSPF dynamic routing
- Routing redundancy and failover
- Centralized DHCP and DHCP relay
- Centralized DNS and internal name resolution
- Dedicated infrastructure services
- Internet connectivity and source NAT
- VLAN segmentation
- 802.1Q trunking
- Router-on-a-stick inter-VLAN routing
- Firewalling and ACL-style policy enforcement
- VPN connectivity
- Python network automation
- Ansible configuration management
- Network monitoring
- IDS/IPS
- Wireshark packet analysis

## Environment

- Windows 11 Pro
- Hyper-V
- GNS3 2.2.61
- GNS3 VM
- KVM hardware acceleration
- VyOS virtual routers
- Ubuntu Server VM (`DHCP01`)
- Kea DHCPv4
- BIND9 DNS
- VPCS endpoint hosts
- GNS3 NAT node
- Wireshark
- GitHub for project documentation and version control

## Repository Structure

```text
GNS3-Network-Security-Homelab/
├── automation/
│   ├── ansible/
│   │   └── README.md
│   └── python/
│       └── README.md
├── configs/
│   └── README.md
├── docs/
│   ├── project-logs/
│   ├── addressing-plan.md
│   ├── design-decisions.md
│   └── troubleshooting.md
├── monitoring/
│   └── README.md
├── packet-captures/
│   └── README.md
├── topology/
│   ├── README.md
│   └── Starting-Topology.png
├── .gitattributes
├── .gitignore
└── README.md
```

### Directory Purpose

- `automation/` — Python and Ansible network automation work.
- `configs/` — Saved or sanitized device and service configurations.
- `docs/` — Addressing plans, design decisions, troubleshooting notes, and dated project logs.
- `monitoring/` — Network monitoring configuration and documentation.
- `packet-captures/` — Wireshark/packet-analysis artifacts and documentation.
- `topology/` — Topology documentation, diagrams, routing-table screenshots, and current GNS3 topology images.

The `docs/project-logs/` directory contains chronological notes documenting implementation, testing, troubleshooting, and lessons learned during each work session.

---

## Current Topology

The current lab contains three VyOS routers, three Layer 2 switches, nine VPCS clients, a dedicated Ubuntu infrastructure server, redundant OSPF paths, six client VLANs, centralized DHCP and DNS, and Internet connectivity through a GNS3 NAT node.

![Current Topology](topology/Centralized%20DHCP%20and%20DNS%20Topology.png)

Current topology diagrams and topology-specific details are available in [`topology/README.md`](topology/README.md).

The lab uses `10.10.0.0/16` as its overall internal RFC1918 private address space.

### VLAN Summary

| VLAN | Subnet | Default Gateway | Clients |
|---|---|---|---|
| 10 | `10.10.10.0/24` | `10.10.10.1` | PC1, PC7 |
| 20 | `10.10.20.0/24` | `10.10.20.1` | PC4 |
| 40 | `10.10.40.0/24` | `10.10.40.1` | PC2, PC8 |
| 50 | `10.10.50.0/24` | `10.10.50.1` | PC3, PC9 |
| 60 | `10.10.60.0/24` | `10.10.60.1` | PC5 |
| 70 | `10.10.70.0/24` | `10.10.70.1` | PC6 |

### Addressing Summary

| Device | Interface | IP Address / Assignment | Purpose |
|---|---|---|---|
| R1 | `eth0.10` | `10.10.10.1/24` | VLAN 10 gateway / DHCP relay |
| R1 | `eth0.40` | `10.10.40.1/24` | VLAN 40 gateway / DHCP relay |
| R1 | `eth0.50` | `10.10.50.1/24` | VLAN 50 gateway / DHCP relay |
| R1 | `eth1` | `10.10.100.1/30` | R1-R2 transit |
| R1 | `eth2` | `10.10.100.9/30` | Direct R1-R3 transit / preferred path |
| R2 | `eth0` | `10.10.100.2/30` | R1-R2 transit |
| R2 | `eth1` | `10.10.100.5/30` | R2-R3 transit |
| R2 | `eth2` | `10.10.30.1/24` | Server-network gateway / OSPF passive interface |
| R2 | `eth3` | DHCP from GNS3 NAT | NAT-facing Internet/WAN interface |
| DHCP01 | `ens3` | `10.10.30.10/24` | Kea DHCP + BIND9 DNS |
| R3 | `eth0` | `10.10.100.6/30` | R2-R3 transit |
| R3 | `eth1.20` | `10.10.20.1/24` | VLAN 20 gateway / DHCP relay |
| R3 | `eth1.60` | `10.10.60.1/24` | VLAN 60 gateway / DHCP relay |
| R3 | `eth1.70` | `10.10.70.1/24` | VLAN 70 gateway / DHCP relay |
| R3 | `eth2` | `10.10.100.10/30` | Direct R1-R3 transit / preferred path |
| PC1 | `eth0` | DHCP: `10.10.10.100/24` current lease | VLAN 10 client |
| PC2 | `eth0` | DHCP: `10.10.40.100/24` current lease | VLAN 40 client |
| PC3 | `eth0` | DHCP: `10.10.50.100/24` current lease | VLAN 50 client |
| PC4 | `eth0` | DHCP: `10.10.20.100/24` current lease | VLAN 20 client |
| PC5 | `eth0` | DHCP: `10.10.60.100/24` current lease | VLAN 60 client |
| PC6 | `eth0` | DHCP: `10.10.70.100/24` current lease | VLAN 70 client |
| PC7 | `eth0` | DHCP: `10.10.10.101/24` current lease | VLAN 10 client |
| PC8 | `eth0` | DHCP: `10.10.40.101/24` current lease | VLAN 40 client |
| PC9 | `eth0` | DHCP: `10.10.50.101/24` current lease | VLAN 50 client |

Core and infrastructure networks:

- `10.10.30.0/24` — server network / DHCP01
- `10.10.100.0/30` — R1-R2 transit
- `10.10.100.4/30` — R2-R3 transit
- `10.10.100.8/30` — direct R1-R3 transit
- `192.168.42.0/24` — GNS3 NAT-facing network

Observed during testing:

```text
GNS3 NAT gateway: 192.168.42.1
R2 eth3:          192.168.42.234/24
```

R2 `eth3` is dynamically assigned by the GNS3 NAT node and is not guaranteed to keep the same address.

See [`docs/addressing-plan.md`](docs/addressing-plan.md) for the full addressing plan.

---

## Completed Milestones

### 1. GNS3 and Hyper-V Environment Setup

- Installed and configured GNS3 Desktop.
- Deployed the GNS3 VM on Hyper-V.
- Enabled nested virtualization for the GNS3 VM.
- Verified KVM hardware acceleration inside the GNS3 VM.
- Created a reusable VyOS QEMU router template.
- Configured VPCS hosts for endpoint testing.

### 2. IPv4 Addressing and Static Routing

Built the initial three-router topology and manually configured IPv4 addresses on every router interface and endpoint.

Static routes were added so endpoint LANs could communicate across the routed topology.

Connectivity was verified using:

- `ping`
- `traceroute`
- `show ip route`
- Directly connected interface testing

This stage demonstrated that each router needs valid routing information for remote networks and a reachable next hop.

### 3. OSPF Dynamic Routing

Static routes were replaced with OSPFv2.

All three VyOS routers participate in backbone Area 0.

Current OSPF-facing interfaces include:

| Router | Router ID | OSPF Interfaces |
|---|---|---|
| R1 | `1.1.1.1` | `eth1`, `eth2`, `eth0.10`, `eth0.40`, `eth0.50` |
| R2 | `2.2.2.2` | `eth0`, `eth1`, `eth2` |
| R3 | `3.3.3.3` | `eth0`, `eth2`, `eth1.20`, `eth1.60`, `eth1.70` |

The VLAN subinterfaces on R1 and R3 are configured as OSPF passive interfaces. R2 `eth2` is also passive.

The parent trunk interfaces `R1 eth0` and `R3 eth1` no longer participate directly in OSPF.

Detailed routing-table screenshots and OSPF information are available in [`topology/README.md`](topology/README.md).

### 4. OSPF Redundancy and Failover

A third router-to-router transit network was added between R1 and R3:

```text
10.10.100.8/30
```

Addresses:

```text
R1 eth2: 10.10.100.9/30
R3 eth2: 10.10.100.10/30
```

With all links operational, OSPF prefers the direct R1-R3 path for traffic crossing between the R1-side and R3-side VLANs.

When R1 `eth2` was disabled, OSPF automatically reconverged through R2:

```text
R1 -> R2 -> R3
```

Connectivity remained available without adding a manual backup route.

After restoring the link, OSPF automatically returned to the preferred direct path.

### 5. DHCP and DHCP Relay: Initial Router-Based Design

DHCP was first tested locally and then centralized on R1.

R1 provided scopes for endpoint networks while R3 acted as a relay for remote clients.

This stage introduced:

- DHCP scopes
- DHCP DORA
- DHCP relay
- Kea listen-address behavior
- Application-level troubleshooting
- DHCP operation across routed networks

During troubleshooting, PC1 DHCP broadcasts were visible in Wireshark but were not processed by Kea as expected. This became an important design driver for moving DHCP to a dedicated Ubuntu server.

The exact Kea root cause was not proven, so the project records this as a troubleshooting hypothesis rather than a confirmed software defect.

### 6. Dedicated Centralized DHCP Server

The DHCP architecture was redesigned so routers no longer host the centralized DHCP service.

A dedicated Ubuntu Server VM was deployed:

```text
Hostname: dhcp01
IP:       10.10.30.10/24
Gateway:  10.10.30.1
Service:  Kea DHCPv4
```

The server network is:

```text
10.10.30.0/24
```

R2 `eth2` acts as its gateway:

```text
10.10.30.1/24
```

Kea now provides six DHCP scopes:

| VLAN | Pool | Gateway | DNS |
|---|---|---|---|
| 10 | `10.10.10.100` to `10.10.10.199` | `10.10.10.1` | `10.10.30.10` |
| 20 | `10.10.20.100` to `10.10.20.199` | `10.10.20.1` | `10.10.30.10` |
| 40 | `10.10.40.100` to `10.10.40.199` | `10.10.40.1` | `10.10.30.10` |
| 50 | `10.10.50.100` to `10.10.50.199` | `10.10.50.1` | `10.10.30.10` |
| 60 | `10.10.60.100` to `10.10.60.199` | `10.10.60.1` | `10.10.30.10` |
| 70 | `10.10.70.100` to `10.10.70.199` | `10.10.70.1` | `10.10.30.10` |

Lease timers:

```text
Valid lifetime: 86400 seconds
Renew timer:    43200 seconds
Rebind timer:   75600 seconds
```

R1 relays DHCP for VLANs `10`, `40`, and `50`.

R3 relays DHCP for VLANs `20`, `60`, and `70`.

All nine clients successfully obtained DHCP leases from DHCP01.

### 7. GNS3 NAT and Internet Connectivity

A GNS3 NAT node was added and connected to R2 `eth3`.

R2 `eth3` acts as a DHCP client and receives its outside-facing address from the GNS3 NAT node.

Observed during testing:

```text
R2 eth3:     192.168.42.234/24
NAT gateway: 192.168.42.1
```

R2 received a default route through the NAT node.

Source NAT masquerading was configured for:

```text
10.10.0.0/16
```

R2 therefore acts as the Internet edge router for all internal VLANs and infrastructure networks.

### 8. OSPF Default-Route Propagation

R2 originates its default route into OSPF:

```text
set protocols ospf default-information originate
```

This allows R1 and R3 to learn that Internet-bound traffic should be forwarded toward R2.

After source NAT and default-route propagation were configured, client VLANs could successfully reach the Internet.

### 9. VLAN Segmentation and Router-on-a-Stick

The lab was expanded from simple endpoint LANs into six VLANs.

R1 provides Layer 3 gateways for:

```text
VLAN 10 -> 10.10.10.1/24
VLAN 40 -> 10.10.40.1/24
VLAN 50 -> 10.10.50.1/24
```

R3 provides Layer 3 gateways for:

```text
VLAN 20 -> 10.10.20.1/24
VLAN 60 -> 10.10.60.1/24
VLAN 70 -> 10.10.70.1/24
```

R1 uses `eth0` as an 802.1Q trunk parent and R3 uses `eth1`.

Layer 2 topology:

```text
R1
 |
Switch1 -------- Switch3
 |                 |
PC1/PC2/PC3      PC7/PC8/PC9
```

Switch1 and Switch3 share VLANs `10`, `40`, and `50` across an 802.1Q trunk.

R3 connects to Switch2, which carries VLANs `20`, `60`, and `70`.

This stage demonstrated the difference between Layer 2 VLAN separation and Layer 3 routing between VLANs.

### 10. Centralized DNS with BIND9

BIND9 was installed on DHCP01.

```text
DNS Server:    10.10.30.10
Internal Zone: gns3.lab
```

All DHCP scopes now distribute:

```text
10.10.30.10
```

as the DNS server.

BIND9 forwards unknown external queries to:

```text
8.8.8.8
1.1.1.1
```

and hosts an authoritative internal `gns3.lab` zone.

Current internal DNS records include:

```text
dhcp01.gns3.lab -> 10.10.30.10
r1.gns3.lab     -> 10.10.100.1
r2.gns3.lab     -> 10.10.100.2
r3.gns3.lab     -> 10.10.100.6
```

Testing confirmed resolution of both internal names and public Internet names from client VLANs.

### 11. VLAN-Aware Routing and Service Failover Validation

The direct R1-R3 link was disabled again after the VLAN, DHCP, DNS, and Internet configuration was complete.

During the failure:

- OSPF reconverged through R2.
- R1-side VLANs could still communicate with R3-side VLANs.
- DHCP relay continued working.
- DHCP01 remained reachable.
- DNS remained available.
- Internet connectivity remained available through R2.

After restoring the link, OSPF returned to the preferred direct R1-R3 path.

This demonstrated that the redundant routed core protects more than simple ICMP connectivity: centralized infrastructure services also remain reachable during a core-link failure.

---

## Troubleshooting Highlights

Several significant issues were investigated during the project, including:

- VMware / Hyper-V conflicts
- Windows Smart App / publisher verification issues affecting GNS3
- OSPF configuration-style conflicts
- OSPF path selection and convergence
- Stale parent interfaces remaining in OSPF after router-on-a-stick conversion
- DHCP relay and Kea listen-address behavior
- PC1 DHCP Discover troubleshooting with Wireshark
- Ubuntu Netplan configuration
- Kea configuration syntax and permissions
- 802.1Q trunk mismatch between Switch1 and Switch3
- Incorrect client addressing during VLAN migration
- Inter-VLAN routing behavior
- Source NAT / masquerading
- Missing OSPF default route for Internet-bound clients
- BIND9 configuration and validation
- Internal and external DNS resolution testing
- Full routing and service failover validation

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for the detailed troubleshooting record.

---

## Current Status

**Current milestone complete: VLAN segmentation, centralized DHCP/DNS, OSPF redundancy, and Internet connectivity**

The network currently has:

- IPv4 addressing and subnetting
- Six client VLANs
- Nine VPCS clients
- Three Layer 2 switches
- 802.1Q trunking
- Router-on-a-stick on R1 and R3
- Dedicated `10.10.30.0/24` server network
- Three routed `/30` core transit networks
- OSPFv2 Area 0
- Passive OSPF VLAN/server-facing interfaces
- Dynamic route learning
- Redundant Layer 3 paths
- Automatic OSPF reconvergence
- Dedicated Ubuntu infrastructure server
- Centralized Kea DHCP
- Six DHCP scopes
- DHCP relay on R1 and R3
- Centralized BIND9 DNS
- Internal `gns3.lab` DNS zone
- External DNS forwarding
- GNS3 NAT Internet connectivity
- R2 source NAT masquerading
- OSPF default-route origination
- Internet access from client VLANs
- Verified DHCP, DNS, routing, and Internet availability during R1-R3 link failure
- Wireshark and service-log troubleshooting experience

One important limitation remains: the VLANs are separate Layer 2 broadcast domains, but inter-VLAN routing is currently permitted by the routers.

---

## Next Milestone

### Inter-VLAN Firewall and Security Policy

The next stage will enforce security policy between VLANs using VyOS firewall rules / ACL-style controls.

Planned goals:

1. Define which VLANs should be allowed to communicate.
2. Block unnecessary inter-VLAN traffic.
3. Preserve access to shared infrastructure services such as DHCP01 and DNS.
4. Preserve required Internet access.
5. Test allowed and denied traffic with `ping`, DNS queries, and application traffic.
6. Document firewall rule order, stateful behavior, and troubleshooting.
7. Validate that routing redundancy continues to work with security policy enabled.

After inter-VLAN security controls are complete, the lab can progress into VPNs, automation, monitoring, and IDS/IPS.
