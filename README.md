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

The current routed topology consists of two endpoint hosts and three VyOS routers:

```text
PC1 --- R1 --- R2 --- R3 --- PC2
```

The lab uses `10.10.0.0/16` as its overall RFC1918 private address space.

### Addressing Summary

| Device | Interface | IP Address | Purpose |
|---|---|---|---|
| PC1 | eth0 | `10.10.10.10/24` | Left-side LAN host |
| R1 | eth0 | `10.10.10.1/24` | PC1 default gateway |
| R1 | eth1 | `10.10.100.1/30` | R1-R2 transit |
| R2 | eth0 | `10.10.100.2/30` | R1-R2 transit |
| R2 | eth1 | `10.10.100.5/30` | R2-R3 transit |
| R3 | eth0 | `10.10.100.6/30` | R2-R3 transit |
| R3 | eth1 | `10.10.20.1/24` | PC2 default gateway |
| PC2 | eth0 | `10.10.20.10/24` | Right-side LAN host |

Current subnets:

- `10.10.10.0/24` — PC1 LAN
- `10.10.100.0/30` — R1-R2 transit
- `10.10.100.4/30` — R2-R3 transit
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
| R1 | `1.1.1.1` | eth0, eth1 |
| R2 | `2.2.2.2` | eth0, eth1 |
| R3 | `3.3.3.3` | eth0, eth1 |

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

## Documentation

Project documentation is separated by purpose:

```text
docs/
├── project-logs/
├── addressing-plan.md
├── design-decisions.md
└── troubleshooting.md

topology/
├── README.md
├── Starting-Topology.png
├── R1-OSPF-ROUTING-TABLE.png
├── R2-OSPF-ROUTING-TABLE.png
└── R3-OSPF-ROUTING-TABLE.png
```

The `project-logs` directory contains chronological notes documenting implementation, testing, troubleshooting, and lessons learned during each work session.

## Current Status

**Milestone complete: Single-area OSPF dynamic routing**

The current network has:

- Working IPv4 addressing and subnetting
- Two `/24` LAN networks
- Two `/30` router transit networks
- End-to-end PC1-to-PC2 connectivity
- Verified static-routing implementation
- Static routes successfully replaced with OSPF
- Full OSPF adjacencies between neighboring routers
- Dynamically learned OSPF routes
- Successful ping and traceroute verification
- Documented addressing, topology, routing tables, and project logs

## Next Milestone

### OSPF Redundancy and Failover

The next step is to add a redundant link between R1 and R3, creating an alternate path through the topology.

The goal is to:

1. Create and address the new R1-R3 transit link.
2. Add the link to OSPF Area 0.
3. Observe OSPF path selection and route costs.
4. Simulate a link failure.
5. Verify that OSPF reconverges and automatically selects the alternate path.
6. Document the before-and-after routing tables and traceroute results.

After the OSPF redundancy milestone, the lab will expand into DHCP, VLAN segmentation, inter-VLAN routing, firewalling, automation, monitoring, and security services.
