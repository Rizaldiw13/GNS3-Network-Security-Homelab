# GNS3 Network Security Homelab

A mixed-vendor network engineering and cybersecurity homelab built in GNS3 to practice routing, switching, centralized infrastructure services, firewall segmentation, troubleshooting, and enterprise-style network design.

The lab has evolved from a small static-routing topology into a segmented network with a redundant OSPF core, centralized DHCP/DNS, pfSense internal firewalls, and a FortiGate perimeter firewall.

## Current Topology

![Current GNS3 Firewall Topology](topology/Current-GNS3-Topology.jpg)

The current design has three main layers:

- **Perimeter security:** FortiGate between the routed core and the GNS3 NAT/Internet.
- **Routed core:** three VyOS routers running OSPF Area 0 with redundant paths.
- **Internal segmentation:** pfSense-Internal protects IT, Finance, and HR; pfSense-Guest protects the Guest network.

```text
                           Internet
                              |
                          GNS3 NAT
                              |
                          FortiGate
                              |
                       10.10.254.0/30
                              |
                              R2
                            /    \
                           /      \
                         R1--------R3
                          |        |
                10.10.252.0/30   10.10.253.0/30
                          |        |
                pfSense-Internal  pfSense-Guest
                          |        |
                        Switch1   Switch2
                          |
                       Switch3
```

DHCP01/DNS01 connects directly to R2 on the `10.10.30.0/24` server network.

See [topology/README.md](topology/README.md) for the full topology documentation and historical evolution.

---

## Technologies

### Virtualization and Lab Platform

- GNS3
- GNS3 VM
- Hyper-V
- QEMU/KVM
- Windows 11 Pro

### Networking and Security

- VyOS
- pfSense CE
- FortiGate
- OSPFv2
- Static routing
- Route redistribution
- IEEE 802.1Q VLAN trunking
- Stateful firewall policy
- DHCP relay
- NAT
- ICMP and routing troubleshooting

### Infrastructure Services

- Ubuntu Server
- Kea DHCPv4
- BIND9 DNS
- Internal `gns3.lab` DNS zone
- Linux Webterm endpoints
- VPCS endpoints

### Project Tooling

- Wireshark
- Git / GitHub
- GNS3 packet capture
- CLI-based router configuration

---

## Current Network Segments

The lab primarily uses `10.10.0.0/16` as its internal RFC1918 address space.

| Network | Role | Default Gateway |
|---|---|---|
| `10.10.10.0/24` | IT / VLAN 10 | `10.10.10.1` on pfSense-Internal |
| `10.10.20.0/24` | Guest | `10.10.20.1` on pfSense-Guest |
| `10.10.30.0/24` | Infrastructure server network | `10.10.30.1` on R2 |
| `10.10.40.0/24` | Finance / VLAN 40 | `10.10.40.1` on pfSense-Internal |
| `10.10.50.0/24` | HR / VLAN 50 | `10.10.50.1` on pfSense-Internal |
| `10.10.100.0/30` | R1-R2 OSPF transit | R1 `.1`, R2 `.2` |
| `10.10.100.4/30` | R2-R3 OSPF transit | R2 `.5`, R3 `.6` |
| `10.10.100.8/30` | R1-R3 OSPF transit | R1 `.9`, R3 `.10` |
| `10.10.252.0/30` | R1-pfSense-Internal transit | R1 `.1`, pfSense `.2` |
| `10.10.253.0/30` | R3-pfSense-Guest transit | R3 `.1`, pfSense `.2` |
| `10.10.254.0/30` | R2-FortiGate transit | FortiGate `.1`, R2 `.2` |

For the full interface-level addressing plan, see [docs/addressing-plan.md](docs/addressing-plan.md).

---

## Core Routing Design

R1, R2, and R3 form an OSPF Area 0 triangle:

```text
                         R2
                       /    \
                      /      \
                    R1--------R3
```

The core transit networks are:

- R1-R2: `10.10.100.0/30`
- R2-R3: `10.10.100.4/30`
- R1-R3: `10.10.100.8/30`

This design provides redundant Layer 3 paths and allows OSPF to reconverge after a core-link failure.

The protected user networks are not directly connected to the VyOS routers anymore.

R1 uses static routes through pfSense-Internal:

```text
10.10.10.0/24 -> 10.10.252.2
10.10.40.0/24 -> 10.10.252.2
10.10.50.0/24 -> 10.10.252.2
```

R3 uses a static route through pfSense-Guest:

```text
10.10.20.0/24 -> 10.10.253.2
```

These protected-network routes are selectively redistributed into OSPF so the rest of the routed core can reach them without making the pfSense firewalls OSPF participants.

---

## Centralized DHCP and DNS

A dedicated Ubuntu server provides both DHCP and DNS:

```text
Hostname: DHCP01
Address:  10.10.30.10/24
Gateway:  10.10.30.1

Services:
- Kea DHCPv4
- BIND9 DNS
```

Current DHCP scopes:

| Network | Pool | Gateway | DNS |
|---|---|---|---|
| IT | `10.10.10.100-199` | `10.10.10.1` | `10.10.30.10` |
| Guest | `10.10.20.100-199` | `10.10.20.1` | `10.10.30.10` |
| Finance | `10.10.40.100-199` | `10.10.40.1` | `10.10.30.10` |
| HR | `10.10.50.100-199` | `10.10.50.1` | `10.10.30.10` |

DHCP relay is performed by the pfSense gateways:

- pfSense-Internal relays IT, Finance, and HR DHCP requests.
- pfSense-Guest relays Guest DHCP requests.
- All relays forward to `10.10.30.10`.

BIND9 provides recursive DNS for the lab and hosts the internal `gns3.lab` zone.

---

## Firewall and Segmentation Design

### pfSense-Internal

pfSense-Internal is the Layer 3 gateway and stateful segmentation firewall for:

- IT — `10.10.10.1/24`
- Finance — `10.10.40.1/24`
- HR — `10.10.50.1/24`

Current policy behavior:

| Source | Destination | Result |
|---|---|---|
| IT | Finance | Allow |
| IT | HR | Allow |
| IT | Guest | Allow |
| IT | pfSense-Internal HTTPS management | Allow |
| Finance | IT | Block |
| Finance | HR | Block |
| HR | IT | Block |
| HR | Finance | Block |
| IT / Finance / HR | DNS01 | Allow DNS |
| IT / Finance / HR | Internet | Allow |

Because pfSense is stateful, return traffic for sessions initiated by permitted sources is allowed automatically.

### pfSense-Guest

pfSense-Guest protects `10.10.20.0/24`.

Guest clients are allowed Internet access but cannot initiate connections toward protected internal/private networks.

IT is explicitly permitted to reach the Guest network for support and troubleshooting.

### FortiGate

FortiGate is the perimeter firewall between R2 and the GNS3 NAT node.

The current Internet path is:

```text
Client
  |
pfSense
  |
R1 or R3
  |
R2
  |
FortiGate
  |
GNS3 NAT
  |
Internet
```

R2 uses FortiGate as its static default gateway and originates the default route into OSPF for R1 and R3.

---

## IT Management Access

The old pfSense-Internal `192.168.1.1/24` LAN/management interface has been removed.

Firewall management now occurs directly through the IT VLAN:

```text
pfSense-Internal GUI: https://10.10.10.1
Allowed source:       10.10.10.0/24
Protocol:             HTTPS / TCP 443
```

`webterm-IT` is a DHCP-based IT workstation used for browser and application-layer testing. It can access the pfSense GUI under the same policy as other IT clients.

---

## Verified Functionality

The current lab has been tested for:

- OSPF adjacency formation and redundant core routing
- OSPF reconvergence after link failure
- Centralized DHCP on IT, Finance, HR, and Guest
- DHCP relay through pfSense
- Centralized DNS through `10.10.30.10`
- Internal and public DNS resolution
- Internet connectivity through FortiGate
- IT-to-Finance connectivity
- IT-to-HR connectivity
- IT-to-Guest support connectivity
- Finance/HR isolation
- Guest isolation from protected internal networks
- IT-only HTTPS management of pfSense-Internal
- Removal of the obsolete pfSense `192.168.1.0/24` management network

---

## Project Evolution

The lab was intentionally built in stages so each networking concept could be implemented and tested before adding more complexity.

Completed stages include:

1. GNS3 and Hyper-V environment setup
2. IPv4 addressing and static routing
3. OSPF dynamic routing
4. OSPF redundancy and failover
5. Initial router-based DHCP testing
6. Dedicated centralized Kea DHCP server
7. GNS3 NAT and Internet connectivity
8. OSPF default-route propagation
9. VLAN segmentation and router-on-a-stick
10. Centralized BIND9 DNS
11. VLAN-aware routing and service failover validation
12. FortiGate perimeter firewall deployment
13. pfSense-Guest isolation
14. pfSense-Internal gateway migration and segmentation
15. DHCP relay migration from VyOS to pfSense
16. Selective static-route redistribution into OSPF
17. IT-to-Guest administrative access
18. IT-based pfSense management and retirement of the old management LAN

Older diagrams and configurations are retained where useful to show the evolution of the design.

---

## Repository Structure

```text
GNS3-Network-Security-Homelab/
├── GNS3-Homelab/        # GNS3 project files
├── automation/          # Planned/ongoing Python and Ansible work
├── configs/             # Router and infrastructure configuration backups
├── docs/
│   ├── project-logs/    # Chronological implementation notes
│   ├── addressing-plan.md
│   ├── design-decisions.md
│   └── troubleshooting.md
├── monitoring/          # Monitoring work
├── packet-captures/     # Packet-analysis artifacts
├── topology/
│   ├── README.md
│   ├── Current-GNS3-Topology.jpg
│   └── historical topology/routing images
└── README.md
```

---

## Key Documentation

- [Current topology and routing design](topology/README.md)
- [Addressing plan](docs/addressing-plan.md)
- [Troubleshooting notes](docs/troubleshooting.md)
- [Design decisions](docs/design-decisions.md)
- [Saved configurations](configs/README.MD)
- [Project logs](docs/project-logs/)

---

## Next Milestones

The firewall-segmentation milestone is complete.

Planned future work includes:

- Firewall-policy documentation and configuration backups
- Network monitoring and alerting
- Python network automation
- Ansible configuration management
- VPN implementation
- IDS/IPS integration
- Expanded packet-capture analysis
- Fault-injection and incident-style troubleshooting scenarios

The long-term goal is to keep evolving the lab toward a realistic small-enterprise environment that can be used for both network-engineering practice and security troubleshooting.
