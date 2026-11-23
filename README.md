# GNS3 Network Security Homelab

A mixed-vendor network engineering and cybersecurity homelab built in GNS3 to practice enterprise-style routing, segmentation, centralized services, firewall policy, troubleshooting, and secure network design.

The current architecture uses **VyOS**, **pfSense**, **FortiGate**, **Kea DHCP**, **BIND9 DNS**, **VPCS/Webterm endpoints**, and a **GNS3 NAT** Internet edge, with the GNS3 VM running on **Hyper-V**.

## Current Topology

![Current GNS3 Firewall Topology](topology/Current-Topology.png)

The current design is divided into three layers:

- **Perimeter security:** FortiGate between the routed core and the Internet/GNS3 NAT.
- **Routed core:** R1, R2, and R3 running OSPF Area 0 with redundant paths.
- **Internal segmentation:** pfSense-Internal protects IT, Finance, and HR; pfSense-Guest protects the Guest network.

Detailed topology documentation is available in [`topology/README.md`](topology/README.md).

---

## Current Architecture

```text
                           Internet
                              |
                          GNS3 NAT
                              |
                          FortiGate
                              |
                             R2
                           /    \
                         R1------R3
                         |        |
                pfSense-Internal pfSense-Guest
                         |        |
                      Switch1   Switch2
                         |
                      Switch3
```

DHCP01/DNS01 is attached to R2 on the dedicated `10.10.30.0/24` server network.

### Protected Networks

| Network | Role | Default Gateway |
|---|---|---|
| `10.10.10.0/24` | IT / VLAN 10 | `10.10.10.1` on pfSense-Internal |
| `10.10.20.0/24` | Guest | `10.10.20.1` on pfSense-Guest |
| `10.10.30.0/24` | Server network | `10.10.30.1` on R2 |
| `10.10.40.0/24` | Finance / VLAN 40 | `10.10.40.1` on pfSense-Internal |
| `10.10.50.0/24` | HR / VLAN 50 | `10.10.50.1` on pfSense-Internal |

### Core / Firewall Transit Networks

| Network | Purpose |
|---|---|
| `10.10.100.0/30` | R1-R2 |
| `10.10.100.4/30` | R2-R3 |
| `10.10.100.8/30` | R1-R3 |
| `10.10.252.0/30` | R1-pfSense-Internal |
| `10.10.253.0/30` | R3-pfSense-Guest |
| `10.10.254.0/30` | R2-FortiGate |

The full interface addressing plan is documented in [`docs/addressing-plan.md`](docs/addressing-plan.md).

---

## Routing Design

R1, R2, and R3 form a redundant OSPF Area 0 triangle.

```text
                         R2
                       /    \
                      /      \
                    R1--------R3
```

**R1 to Internal Networks**

The protected client networks are no longer directly connected to R1 or R3.

R1 reaches the IT, Finance, and HR networks behind pfSense-Internal using static routes:

|Destination network|  Next hop|
|---|---|
|`10.10.10.0/24` (IT) |         via `10.10.252.2`|
|`10.10.40.0/24` (Finance)|         via `10.10.252.2`|
|`10.10.50.0/24` (HR) |        via `10.10.252.2`|

`10.10.252.2` is the transit/WAN address of pfSense-Internal.

These static routes are selectively redistributed into OSPF so R2 and R3 can also learn how to reach the IT, Finance, and HR networks.

**R3 to Guest Network**

R3 reaches the Guest network using a static route as well:

|Destination network|  Next hop|
|---|---|
|`10.10.20.0/24` | via `10.10.253.2` | 

`10.10.253.2` is the transit/WAN address of pfSense-Guest.

The Guest route is also selectively redistributed into OSPF so R1 and R2 can learn how to reach the Guest network.

**R2 to FortiGate**

R2 uses FortiGate as its upstream gateway for Internet/North-bound traffic:

Destination network|  Next hop|
|---|---|
|`0.0.0.0/0` | via `10.10.254.1` | 

`10.10.254.1` is FortiGate's internal-facing interface connected to R2.

R2 originates this default route into OSPF so R1 and R3 dynamically learn that Internet-bound traffic should be forwarded toward R2.

---

## VLAN and Switching Design

The internal network uses three active employee VLANs:

| VLAN | Department | Gateway | Example Clients |
|---|---|---|---|
| 10 | IT | `10.10.10.1` | PC1, PC7, webterm-IT |
| 40 | Finance | `10.10.40.1` | PC2, PC8 |
| 50 | HR | `10.10.50.1` | PC3, PC9 |

Switch1 connects to pfSense-Internal using an **802.1Q trunk** carrying VLANs 10, 40, and 50.

Switch1 and Switch3 are joined by another 802.1Q trunk carrying the same VLANs.

The earlier router-on-a-stick design on R1 was retired. pfSense-Internal now owns the Layer 3 gateways and performs inter-VLAN policy enforcement.

The Guest network is a separate `10.10.20.0/24` network behind pfSense-Guest and Switch2.

---

## Centralized DHCP and DNS

DHCP01 provides both DHCP and DNS:

```text
Hostname: DHCP01
IP:       10.10.30.10/24
Gateway:  10.10.30.1

DHCP:     Kea DHCPv4
DNS:      BIND9
```

Current DHCP scopes:

| Network | Pool | Gateway | DNS |
|---|---|---|---|
| IT | `10.10.10.100-199` | `10.10.10.1` | `10.10.30.10` |
| Guest | `10.10.20.100-199` | `10.10.20.1` | `10.10.30.10` |
| Finance | `10.10.40.100-199` | `10.10.40.1` | `10.10.30.10` |
| HR | `10.10.50.100-199` | `10.10.50.1` | `10.10.30.10` |

DHCP relay now runs on the **pfSense firewalls**, not on R1/R3.

BIND9 provides:

- Recursive DNS for internal clients.
- Forwarding for public Internet names.
- Authoritative DNS for the internal `gns3.lab` zone.

---

## Firewall and Security Policy

### pfSense-Internal

pfSense-Internal is the gateway and policy-enforcement point for IT, Finance, and HR.

Verified policy behavior includes:

| Source | Destination | Result |
|---|---|---|
| IT | Finance | Allow |
| IT | HR | Allow |
| IT | Guest | Allow |
| Finance | IT | Block |
| Finance | HR | Block |
| HR | IT | Block |
| HR | Finance | Block |
| IT / Finance / HR | DNS01 | Allow required DNS traffic |
| IT / Finance / HR | Internet | Allow |
| IT | pfSense-Internal HTTPS GUI | Allow |
| IT | pfSense-Guest HTTPS GUI | Allow |
| IT | FortiGate HTTPS GUI | Allow |

All IT clients are permitted to manage pfSense-Internal over:

```text
https://10.10.10.1
TCP/443
```

The obsolete `192.168.1.0/24` pfSense management network was removed after management was moved into the IT VLAN.

### pfSense-Guest

pfSense-Guest protects the Guest network.

Guest clients can reach the Internet but cannot initiate traffic into protected internal networks.

IT is explicitly allowed to reach the Guest network so IT staff can troubleshoot Guest endpoints.

### FortiGate

FortiGate acts as the perimeter firewall between R2 and the GNS3 NAT/Internet.

The active perimeter policy uses the `INTERNAL-NETWORKS` address object:

```text
INTERNAL-NETWORKS = 10.10.0.0/16
```

Outbound traffic from the internal routed network is explicitly permitted for:

- HTTPS
- DNS
- NTP
- ICMP/PING

NAT is enabled on the `Internal-to-Internet` policy. Traffic that does not match an explicit FortiGate allow policy falls through to FortiGate's implicit deny behavior.

FortiGate also has a return route for the internal address space:

```text
10.10.0.0/16 -> via 10.10.254.2 -> port2
```

Trusted IT clients can remotely manage the FortiGate GUI at:

```text
https://10.10.254.1
TCP/443
```

This management path has been verified from `webterm-IT`.

---

## Current Endpoint Layout

| Endpoint | Network |
|---|---|
| PC1 | IT |
| PC2 | Finance |
| PC3 | HR |
| PC4 | Guest |
| PC5 | Guest |
| PC6 | Guest |
| PC7 | IT |
| PC8 | Finance |
| PC9 | HR |
| webterm-IT | IT, DHCP client, browser-enabled workstation |
| webterm-1 | Guest-side browser-enabled workstation |

`webterm-IT` currently receives its address through DHCP and represents an IT administrative workstation. It is used for application-layer testing, web browsing, and HTTPS management of pfSense-Internal, pfSense-Guest, and FortiGate.

---

## Completed Milestones

### 1. Virtual Lab Platform

- Installed and configured GNS3 Desktop.
- Deployed the GNS3 VM on Hyper-V.
- Enabled nested virtualization and KVM acceleration.
- Built reusable VyOS and endpoint templates.

### 2. IPv4 Addressing and Static Routing

- Built the initial three-router network.
- Configured point-to-point and endpoint addressing.
- Verified routing manually using static routes, `ping`, `traceroute`, and routing tables.

### 3. OSPF Dynamic Routing

- Replaced core static routes with OSPFv2 Area 0.
- Built a redundant R1-R2-R3 triangle.
- Tested OSPF reconvergence during core-link failures.

### 4. Centralized Infrastructure Services

- Deployed DHCP01 on Ubuntu Server.
- Implemented Kea DHCPv4.
- Implemented BIND9 DNS.
- Created the `gns3.lab` internal DNS zone.
- Verified internal and public DNS resolution.

### 5. VLAN Segmentation and 802.1Q Trunking

- Built IT, Finance, and HR VLANs.
- Configured Switch1-Switch3 trunking.
- Initially implemented router-on-a-stick to understand inter-VLAN routing.

### 6. Internet Connectivity

- Added GNS3 NAT connectivity.
- Implemented default-route propagation.
- Verified Internet access and DNS from client networks.

### 7. Firewall Architecture Migration

The lab was redesigned from router-based inter-VLAN routing into a firewall-based architecture:

- Added **pfSense-Internal** between R1 and the employee VLANs.
- Added **pfSense-Guest** between R3 and the Guest network.
- Added **FortiGate** between R2 and GNS3 NAT.
- Moved client gateway ownership from R1/R3 to pfSense.
- Removed obsolete R1 VLAN subinterfaces.
- Moved DHCP relay to pfSense.
- Added static firewall-facing routes and selective OSPF redistribution.
- Removed the obsolete `192.168.1.0/24` pfSense management network.

### 8. Security Policy Enforcement

- Isolated Finance and HR from each other.
- Prevented Finance/HR from initiating traffic to IT.
- Isolated Guest from protected internal networks.
- Allowed IT administrative access to Finance, HR, and Guest.
- Restricted pfSense-Internal GUI management to IT clients over HTTPS.
- Added trusted IT HTTPS management of pfSense-Guest.
- Added trusted IT HTTPS management of FortiGate at `10.10.254.1`.
- Enforced FortiGate perimeter egress filtering for HTTPS, DNS, NTP, and PING with implicit deny for unmatched traffic.
- Preserved centralized DHCP, DNS, and Internet access.

The **firewall-segmentation milestone is now complete!**

---

## Troubleshooting Highlights

The lab has included hands-on troubleshooting of:

- Hyper-V / virtualization compatibility
- OSPF adjacency and path selection
- OSPF failover and reconvergence
- VLAN trunk mismatches
- DHCP relay and Kea behavior
- DHCP broadcast analysis with Wireshark
- Ubuntu network configuration
- BIND9 validation and DNS forwarding
- Source NAT and default-route issues
- pfSense VLAN migration
- DHCP relay downstream-interface selection
- Static-route redistribution into OSPF
- Guest isolation and IT-to-Guest access
- FortiGate-dependent Internet reachability
- Webterm persistent IP/network configuration
- Temporary pfSense GUI lockout during management-network migration
- Recovery using the pfSense console and `pfctl`
- Multi-firewall management-path troubleshooting
- FortiGate HTTPS management from the IT VLAN


See [`docs/troubleshooting.md`](docs/troubleshooting.md) for the detailed troubleshooting record.

---

## Environment and Technologies

- Windows 11
- Hyper-V
- GNS3 Desktop + GNS3 VM
- VyOS
- pfSense CE
- FortiGate VM
- Ubuntu Server
- Kea DHCPv4
- BIND9
- VPCS
- Webterm / lightweight Linux endpoints
- GNS3 NAT
- Wireshark
- Git / GitHub

---

## Repository Structure

```text
GNS3-Network-Security-Homelab/
├── automation/
│   ├── ansible/
│   └── python/
├── configs/
├── docs/
│   ├── project-logs/
│   ├── addressing-plan.md
│   ├── design-decisions.md
│   └── troubleshooting.md
├── monitoring/
├── packet-captures/
├── topology/
│   ├── README.md
│   ├── Current-Topology.png
│   └── historical topology images
└── README.md
```

---

## Project Goals / Future Expansion

The lab is being developed progressively toward a broader enterprise-networking and security environment.

Future areas include:

- VPN connectivity
- Network automation with Python
- Ansible configuration management
- Centralized monitoring and alerting
- IDS/IPS
- Additional packet-capture analysis
- Configuration backup/versioning
- Structured fault-injection and troubleshooting scenarios

The goal is not only to build a functioning network, but also to practice design, implementation, verification, troubleshooting, documentation, and security-policy reasoning across a realistic multi-vendor environment.
