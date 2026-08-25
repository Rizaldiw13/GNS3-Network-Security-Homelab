# GNS3 Network Security Homelab

A mixed-vendor network engineering and cybersecurity homelab built using GNS3, Hyper-V, VyOS, and VPCS.

## Project Goals

This project progressively builds an enterprise-style virtual network covering:

- IPv4 addressing and subnetting
- Static routing
- OSPF dynamic routing
- Routing redundancy and failover
- DHCP and DHCP relay
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
- VPCS endpoint hosts
- GitHub for project documentation and version control

## Current Topology

The current routed topology consists of two endpoint hosts and three VyOS routers with a redundant R1-R3 transit link:

```text
                R2
               /  \
              /    \
PC1 --- R1 -------- R3 --- PC2
```

The lab uses `10.10.0.0/16` as its overall RFC1918 private address space.

### Addressing Summary

| Device | Interface | IP Address | Purpose |
|---|---|---|---|
| PC1 | eth0 | `10.10.10.10/24` | Left-side LAN host |
| R1 | eth0 | `10.10.10.1/24` | PC1 default gateway |
| R1 | eth1 | `10.10.100.1/30` | R1-R2 transit |
| R1 | eth2 | `10.10.100.9/30` | R1-R3 redundant transit |
| R2 | eth0 | `10.10.100.2/30` | R1-R2 transit |
| R2 | eth1 | `10.10.100.5/30` | R2-R3 transit |
| R3 | eth0 | `10.10.100.6/30` | R2-R3 transit |
| R3 | eth1 | `10.10.20.1/24` | PC2 default gateway |
| R3 | eth2 | `10.10.100.10/30` | R1-R3 redundant transit |
| PC2 | eth0 | `10.10.20.10/24` | Right-side LAN host |

Current subnets:

- `10.10.10.0/24` — PC1 LAN
- `10.10.100.0/30` — R1-R2 transit
- `10.10.100.4/30` — R2-R3 transit
- `10.10.100.8/30` — R1-R3 redundant transit
- `10.10.20.0/24` — PC2 LAN

See [`docs/addressing-plan.md`](docs/addressing-plan.md) for the full addressing plan.

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

The static-routing stage was used to understand how each router needs explicit knowledge of remote networks and valid next-hop addresses.

### 3. OSPF Dynamic Routing

The static routes were removed and replaced with OSPFv2.

All routers currently participate in backbone Area 0.

| Router | Router ID | OSPF Interfaces |
|---|---|---|
| R1 | `1.1.1.1` | eth0, eth1, eth2 |
| R2 | `2.2.2.2` | eth0, eth1 |
| R3 | `3.3.3.3` | eth0, eth1, eth2 |

OSPF neighbor relationships successfully reached the `Full` state:

```text
                     Area 0

       1.1.1.1          2.2.2.2          3.3.3.3
          R1 -------------- R2 -------------- R3
                    Full              Full
```

The routers now dynamically learn remote networks through OSPF rather than relying on manually configured static routes.

Examples:

- R1 learns `10.10.20.0/24` through R2.
- R2 learns both remote LANs.
- R3 learns `10.10.10.0/24` through R2.

End-to-end connectivity between PC1 and PC2 has been verified again using both ping and traceroute.

Detailed topology diagrams, routing-table screenshots, and OSPF information are available in [`topology/README.md`](topology/README.md).

### 4. OSPF Redundancy and Failover

Added a third router-to-router transit network between R1 and R3:

`10.10.100.8/30`

- R1 eth2: `10.10.100.9/30`
- R3 eth2: `10.10.100.10/30`

The new link was added to OSPF Area 0, forming a redundant triangle between R1, R2, and R3.

With all links operational, OSPF selected the lower-cost direct path from R1 to R3 for traffic destined for the PC2 LAN:

```text
PC1 -> R1 -> R3 -> PC2
```

R1's route to the PC2 LAN was:

```text
O>* 10.10.20.0/24 [110/2] via 10.10.100.10, eth2
```

To test failover, R1's `eth2` interface was administratively disabled. OSPF detected the topology change, removed the direct R1-R3 adjacency, recalculated the shortest path, and automatically failed traffic over through R2:

```text
PC1 -> R1 -> R2 -> R3 -> PC2
```

During the failure, R1's route changed to:

```text
O>* 10.10.20.0/24 [110/3] via 10.10.100.2, eth1
```

PC1 retained end-to-end connectivity to PC2 throughout the test. After restoring the R1-R3 link, OSPF automatically returned to the lower-cost direct path.

This milestone verified:

- Redundant Layer 3 connectivity
- OSPF path selection
- OSPF cost changes
- Neighbor loss and re-formation
- Dynamic reconvergence
- Automatic failover
- Preferred-path restoration
- Validation using routing tables, ping, and traceroute

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
│   ├── R1-OSPF-ROUTING-TABLE.png
│   ├── R2-OSPF-ROUTING-TABLE.png
│   ├── R3-OSPF-ROUTING-TABLE.png
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
- `topology/` — Topology documentation, diagrams, and routing-table screenshots.

The `docs/project-logs/` directory contains chronological notes documenting implementation, testing, troubleshooting, and lessons learned during each work session.

## Current Status

**Milestone complete: OSPF redundancy and automatic failover**

The current network has:

- Working IPv4 addressing and VLSM-based subnetting
- Two `/24` LAN networks
- Three `/30` router transit networks
- Three VyOS routers running OSPFv2 in Area 0
- Full OSPF adjacencies across the redundant router topology
- Dynamically learned OSPF routes
- A direct R1-R3 redundant transit path
- OSPF shortest-path selection based on route cost
- Verified automatic reconvergence when the preferred R1-R3 link fails
- Continuous PC1-to-PC2 connectivity during the simulated failure
- Automatic restoration of the lower-cost route when the failed link returns
- Verification using routing tables, neighbor tables, ping, and traceroute
- Documentation of addressing, topology, routing behavior, and dated project logs

The current router topology is:

```text
                R2
               /  \
              /    \
PC1 --- R1 -------- R3 --- PC2
```

## Next Milestone

### DHCP and DHCP Relay

The next stage will replace manually configured endpoint addressing with DHCP.

The initial goals are to:

1. Configure DHCP scopes for the endpoint LANs.
2. Automatically provide host IP addresses, subnet masks, and default gateways.
3. Verify DHCP lease assignment and renewal.
4. Introduce a centralized DHCP design.
5. Configure DHCP relay so clients on remote subnets can obtain addresses from a DHCP server located on another network.
6. Document DHCP message flow, configuration, leases, and connectivity testing.

After DHCP, the lab will expand into VLAN segmentation and inter-VLAN routing, including additional switches and endpoint devices.

