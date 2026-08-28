# GNS3 Network Security Homelab

A mixed-vendor network engineering and cybersecurity homelab built using GNS3, Hyper-V, VyOS, Ubuntu Server, Kea DHCP, and VPCS.

## Project Goals

This project progressively builds an enterprise-style virtual network covering:

- IPv4 addressing and subnetting
- Static routing
- OSPF dynamic routing
- Routing redundancy and failover
- Centralized DHCP and DHCP relay
- Dedicated infrastructure services
- Internet connectivity and source NAT
- VLAN segmentation
- Inter-VLAN routing
- Firewalling and NAT
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
- `configs/` — Saved or sanitized device configurations.
- `docs/` — Addressing plans, design decisions, troubleshooting notes, and dated project logs.
- `monitoring/` — Network monitoring configuration and documentation.
- `packet-captures/` — Wireshark/packet-analysis artifacts and documentation.
- `topology/` — Topology documentation, diagrams, routing-table screenshots, and current GNS3 topology images.

The `docs/project-logs/` directory contains chronological notes documenting implementation, testing, troubleshooting, and lessons learned during each work session.

---

## Current Topology

The current lab contains three VyOS routers, two client LANs, a dedicated Ubuntu DHCP server, redundant OSPF paths, and Internet connectivity through a GNS3 NAT node.

![Current-Topology](topology/DHCP-Topology.png)

Current topology diagrams are available in [`topology/README.md`](topology/README.md).

The lab uses `10.10.0.0/16` as its overall internal RFC1918 private address space.

### Addressing Summary

| Device | Interface | IP Address / Assignment | Purpose |
|---|---|---|---|
| PC1 | `eth0` | DHCP: `10.10.10.100/24` current lease | LAN1 client |
| R1 | `eth0` | `10.10.10.1/24` | LAN1 gateway and DHCP relay |
| R1 | `eth1` | `10.10.100.1/30` | R1-R2 transit |
| R1 | `eth2` | `10.10.100.9/30` | Direct R1-R3 transit / preferred path |
| R2 | `eth0` | `10.10.100.2/30` | R1-R2 transit |
| R2 | `eth1` | `10.10.100.5/30` | R2-R3 transit |
| R2 | `eth2` | `10.10.30.1/24` | Server-network gateway / OSPF passive interface |
| R2 | `eth3` | DHCP from GNS3 NAT | NAT-facing Internet/WAN interface |
| DHCP01 | `ens3` | `10.10.30.10/24` | Dedicated Ubuntu Kea DHCP server |
| R3 | `eth0` | `10.10.100.6/30` | R2-R3 transit |
| R3 | `eth1` | `10.10.20.1/24` | LAN2 gateway and DHCP relay |
| R3 | `eth2` | `10.10.100.10/30` | Direct R1-R3 transit / preferred path |
| PC2 | `eth0` | DHCP: `10.10.20.100/24` current lease | LAN2 client |

Current internal subnets:

- `10.10.10.0/24` — LAN1 / PC1
- `10.10.20.0/24` — LAN2 / PC2
- `10.10.30.0/24` — Server network / DHCP01
- `10.10.100.0/30` — R1-R2 transit
- `10.10.100.4/30` — R2-R3 transit
- `10.10.100.8/30` — Direct R1-R3 transit

NAT-facing network observed during testing:

- `192.168.42.0/24` — GNS3 NAT network
- GNS3 NAT gateway: `192.168.42.1`
- R2 `eth3`: DHCP-assigned; observed as `192.168.42.234/24`

The R2 `eth3` address is assigned by the **GNS3 NAT node's DHCP service**, not by `DHCP01`.

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

Static routes were added so PC1 and PC2 could communicate across all three routers.

Connectivity was verified using:

- `ping`
- `traceroute`
- `show ip route`
- Directly connected interface testing

This stage demonstrated that each router needs valid routing information for remote networks and a reachable next hop.

### 3. OSPF Dynamic Routing

Static routes were replaced with OSPFv2.

All three VyOS routers participate in backbone Area 0.

| Router | Router ID | Important OSPF Interfaces |
|---|---|---|
| R1 | `1.1.1.1` | `eth0`, `eth1`, `eth2` |
| R2 | `2.2.2.2` | `eth0`, `eth1`, `eth2` |
| R3 | `3.3.3.3` | `eth0`, `eth1`, `eth2` |

R2 `eth2` connects to the `10.10.30.0/24` server network and is configured as an **OSPF passive interface**.

This allows R2 to advertise the server network while preventing unnecessary OSPF Hello packets and adjacency attempts toward `DHCP01`.

```text
Advertise 10.10.30.0/24 into OSPF: Yes
Form OSPF adjacency on R2 eth2:    No
```

Detailed routing-table screenshots and OSPF information are available in [`topology/README.md`](topology/README.md).

### 4. OSPF Redundancy and Failover

Added a third router-to-router transit network between R1 and R3:

```text
10.10.100.8/30
```

Addresses:

```text
R1 eth2: 10.10.100.9/30
R3 eth2: 10.10.100.10/30
```

With all links operational, OSPF selects the direct R1-R3 path:

```text
PC1 -> R1 -> R3 -> PC2
```

Example R1 route:

```text
O>* 10.10.20.0/24 [110/2] via 10.10.100.10, eth2
```

When R1 `eth2` was disabled, OSPF automatically reconverged through R2:

```text
PC1 -> R1 -> R2 -> R3 -> PC2
```

Example failed-over route:

```text
O>* 10.10.20.0/24 [110/3] via 10.10.100.2, eth1
```

The route cost increased from 2 to 3, and connectivity remained available without adding a manual backup route.

After restoring the link, OSPF automatically returned to the lower-cost direct path.

### 5. DHCP and DHCP Relay — Initial Router-Based Design

DHCP was first tested locally and then centralized on R1.

R1 provided scopes for both endpoint LANs while R3 acted as a relay for LAN2.

This stage introduced:

- DHCP scopes
- DHCP DORA
- DHCP relay
- Kea listen-address behavior
- Application-level troubleshooting
- DHCP operation across routed networks

During troubleshooting, a Kea socket/listen-address issue showed that successful IP routing does not guarantee that a service is correctly bound to the interface selected for the return path.

This router-based DHCP implementation is retained in the project history because it led directly to the current dedicated-server design.

### 6. Dedicated Centralized DHCP Server

The DHCP architecture was redesigned so the routers no longer host the centralized DHCP service.

A dedicated Ubuntu Server VM was deployed:

```text
Hostname: dhcp01
IP:       10.10.30.10/24
Gateway:  10.10.30.1
Service:  Kea DHCPv4
```

A new server subnet was introduced:

```text
10.10.30.0/24
```

R2 `eth2` acts as the gateway:

```text
10.10.30.1/24
```

#### DHCP Scopes on DHCP01

```text
LAN1
Network: 10.10.10.0/24
Pool:    10.10.10.100 to 10.10.10.199
Gateway: 10.10.10.1
```

```text
LAN2
Network: 10.10.20.0/24
Pool:    10.10.20.100 to 10.10.20.199
Gateway: 10.10.20.1
```

Lease timers:

```text
Valid lifetime: 86400 seconds
Renew timer:    43200 seconds
Rebind timer:   75600 seconds
```

DNS is **not configured yet** for the client DHCP scopes.

#### DHCP Relay Roles

R1 relays LAN1 requests:

```text
PC1 -> R1 DHCP Relay -> DHCP01
```

R3 relays LAN2 requests:

```text
PC2 -> R3 DHCP Relay -> DHCP01
```

Both clients successfully completed DORA:

```text
PC1
IP:          10.10.10.100/24
Gateway:     10.10.10.1
DHCP Server: 10.10.30.10
```

```text
PC2
IP:          10.10.20.100/24
Gateway:     10.10.20.1
DHCP Server: 10.10.30.10
```

Kea logs and the lease database were used to verify successful DHCP operation.

### 7. GNS3 NAT and Internet Connectivity

A GNS3 NAT node was added and connected to R2 `eth3`.

R2 `eth3` acts as a DHCP client and receives its outside-facing address from the **GNS3 NAT node**, not from DHCP01.

Observed during testing:

```text
R2 eth3:     192.168.42.234/24
NAT gateway: 192.168.42.1
```

R2 received a default route:

```text
0.0.0.0/0 via 192.168.42.1
```

Initially, R2 could reach the Internet but internal `10.10.x.x` hosts could not.

Source NAT masquerading was added on R2 for:

```text
10.10.0.0/16
```

Conceptually:

```text
Internal host
    |
R1 / R3
    |
   R2
    |
Source NAT / Masquerade
    |
GNS3 NAT
    |
Internet
```

### 8. OSPF Default-Route Propagation

Even after source NAT was working, PC1 and PC2 initially could not reach the Internet because R1 and R3 did not know that R2 was the Internet exit.

R2 was configured to originate its default route into OSPF:

```text
set protocols ospf default-information originate
```

R1 and R3 then learned:

```text
0.0.0.0/0
```

through OSPF toward R2.

After this change:

```text
PC1 -> 8.8.8.8  successful
PC2 -> 8.8.8.8  successful
```

This demonstrated the difference between:

- routing traffic toward the Internet
- translating private source addresses with NAT
- DNS name resolution

### 9. Full Service Failover Validation

The direct R1-R3 link was disabled again after the DHCP redesign and Internet connectivity were complete.

During the failure:

- OSPF reconverged through R2.
- PC1 could still reach PC2.
- PC2 could still reach PC1.
- PC1 could still obtain DHCP from `10.10.30.10`.
- PC2 could still obtain DHCP from `10.10.30.10`.
- PC1 could still reach `8.8.8.8`.
- PC2 could still reach `8.8.8.8`.

After restoring the link, OSPF returned to the preferred direct R1-R3 path.

This validated that the network's routing redundancy also preserves centralized DHCP and Internet access.

---

## Troubleshooting Highlights

Several significant issues were investigated during the project, including:

- VMware / Hyper-V conflicts
- OSPF configuration-style conflicts
- OSPF path selection and convergence
- DHCP relay and Kea listen-address behavior
- PC1 DHCP Discover troubleshooting with Wireshark
- Ubuntu Netplan configuration
- Kea configuration syntax and permissions
- Source NAT / masquerading
- Missing OSPF default route for Internet-bound clients
- Full routing and service failover validation

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for the detailed troubleshooting record.

---

## Current Status

**Current milestone complete: Dedicated centralized DHCP, dynamic routing redundancy, and Internet connectivity**

The network currently has:

- IPv4 addressing and subnetting
- Two DHCP client LANs
- Dedicated `10.10.30.0/24` server network
- Three routed `/30` transit networks
- OSPFv2 Area 0
- Full router-to-router OSPF adjacencies
- Passive OSPF server-facing interface
- Dynamic route learning
- Redundant Layer 3 paths
- Automatic OSPF reconvergence
- Dedicated Ubuntu Kea DHCP server
- DHCP relay on both R1 and R3
- Verified DHCP DORA for PC1 and PC2
- GNS3 NAT Internet connectivity
- R2 source NAT masquerading
- OSPF default-route origination
- Internet access from both client LANs
- Verified DHCP and Internet availability during R1-R3 link failure
- Wireshark and service-log troubleshooting experience

---

## Next Milestone

### VLAN Segmentation and Inter-VLAN Routing

The next stage will introduce managed Layer 2 switching and VLANs so the lab can evolve from simple endpoint LANs into a segmented enterprise-style network.

Planned goals:

1. Add managed Layer 2 switching to the topology.
2. Create multiple VLANs for different host groups or functions.
3. Configure access ports and 802.1Q trunks.
4. Configure inter-VLAN routing.
5. Extend centralized DHCP to the new VLAN subnets.
6. Verify VLAN isolation and inter-VLAN connectivity.
7. Prepare the topology for ACL and firewall policy enforcement.

After VLAN segmentation, the lab can progress into firewalling, VPNs, automation, monitoring, and IDS/IPS.
