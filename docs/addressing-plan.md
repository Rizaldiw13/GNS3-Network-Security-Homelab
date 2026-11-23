# Addressing Plan

**Updated:** 2026-11-20

## 1. Overall Address Space

The lab primarily uses **10.10.0.0/16 (RFC1918)** for internal routed networks.

The GNS3 NAT-facing network is assigned dynamically to the FortiGate WAN interface and is therefore not treated as a fixed part of the internal addressing plan.

## 2. Network Summary

| Network | Prefix | Purpose | Gateway / Key Addresses | Address Assignment |
|---|---:|---|---|---|
| `10.10.10.0` | `/24` | VLAN 10 - IT | Gateway: `10.10.10.1` on pfSense-Internal | DHCP from DHCP01 via pfSense-Internal relay |
| `10.10.20.0` | `/24` | Guest network | Gateway: `10.10.20.1` on pfSense-Guest | DHCP from DHCP01 via pfSense-Guest relay |
| `10.10.30.0` | `/24` | Server network | Gateway: `10.10.30.1` on R2; DHCP01/DNS01: `10.10.30.10` | Static infrastructure addressing |
| `10.10.40.0` | `/24` | VLAN 40 - Finance | Gateway: `10.10.40.1` on pfSense-Internal | DHCP from DHCP01 via pfSense-Internal relay |
| `10.10.50.0` | `/24` | VLAN 50 - HR | Gateway: `10.10.50.1` on pfSense-Internal | DHCP from DHCP01 via pfSense-Internal relay |
| `10.10.100.0` | `/30` | R1-R2 transit | R1 `.1`, R2 `.2` | Static |
| `10.10.100.4` | `/30` | R2-R3 transit | R2 `.5`, R3 `.6` | Static |
| `10.10.100.8` | `/30` | Direct R1-R3 transit | R1 `.9`, R3 `.10` | Static |
| `10.10.252.0` | `/30` | R1-pfSense-Internal transit | R1 `.1`, pfSense-Internal `.2` | Static |
| `10.10.253.0` | `/30` | R3-pfSense-Guest transit | R3 `.1`, pfSense-Guest `.2` | Static |
| `10.10.254.0` | `/30` | R2-FortiGate transit | FortiGate `.1`, R2 `.2` | Static |

## 3. Infrastructure Interface Addressing

| Device | Interface | IPv4 Address / Prefix | Connected To | Purpose / Notes |
|---|---|---|---|---|
| R1 | `eth1` | `10.10.100.1/30` | R2 `eth0` | R1-R2 OSPF transit |
| R1 | `eth2` | `10.10.100.9/30` | R3 `eth2` | Direct R1-R3 OSPF transit |
| R1 | `eth3` | `10.10.252.1/30` | pfSense-Internal WAN | Transit to internal segmentation firewall |
| R2 | `eth0` | `10.10.100.2/30` | R1 `eth1` | R1-R2 OSPF transit |
| R2 | `eth1` | `10.10.100.5/30` | R3 `eth0` | R2-R3 OSPF transit |
| R2 | `eth2` | `10.10.30.1/24` | DHCP01/DNS01 | Server-network gateway; OSPF passive |
| R2 | `eth3` | `10.10.254.2/30` | FortiGate `port2` | Upstream transit toward perimeter firewall |
| R3 | `eth0` | `10.10.100.6/30` | R2 `eth1` | R2-R3 OSPF transit |
| R3 | `eth1` | `10.10.253.1/30` | pfSense-Guest WAN | Transit to Guest firewall |
| R3 | `eth2` | `10.10.100.10/30` | R1 `eth2` | Direct R1-R3 OSPF transit |
| pfSense-Internal | WAN | `10.10.252.2/30` | R1 `eth3` | Upstream routed interface; gateway `10.10.252.1` |
| pfSense-Internal | IT (VLAN 10) | `10.10.10.1/24` | Switch1 trunk | Default gateway for IT |
| pfSense-Internal | Finance (VLAN 40) | `10.10.40.1/24` | Switch1 trunk | Default gateway for Finance |
| pfSense-Internal | HR (VLAN 50) | `10.10.50.1/24` | Switch1 trunk | Default gateway for HR |
| pfSense-Guest | WAN | `10.10.253.2/30` | R3 `eth1` | Upstream routed interface; gateway `10.10.253.1` |
| pfSense-Guest | LAN | `10.10.20.1/24` | Switch2 | Guest default gateway |
| FortiGate | `port2` | `10.10.254.1/30` | R2 `eth3` | Internal/perimeter transit toward R2 |
| FortiGate | WAN / `port1` | DHCP from GNS3 NAT | GNS3 NAT node | Internet-facing interface; dynamic address |
| DHCP01 | `ens3` | `10.10.30.10/24` | R2 `eth2` | Kea DHCPv4 + BIND9 DNS |
| webterm-1 | `eth0` | `10.10.20.250/24` | Switch2 / Guest LAN | Guest-side browser/test workstation; gateway `10.10.20.1` |
| webterm-IT | `eth0` | DHCP on `10.10.10.0/24` (observed `10.10.10.102/24`) | Switch1 VLAN 10 | IT administrative workstation; gateway `10.10.10.1`; DNS `10.10.30.10`; HTTPS management access to pfSense-Internal, pfSense-Guest, and FortiGate |

## 4. VLAN and Client Layout

| VLAN / Network | Department / Role | Gateway | Clients |
|---|---|---|---|
| VLAN 10 - `10.10.10.0/24` | IT | `10.10.10.1` | PC1 and webterm-IT on Switch1; PC7 on Switch3 |
| VLAN 40 - `10.10.40.0/24` | Finance | `10.10.40.1` | PC2 on Switch1; PC8 on Switch3 |
| VLAN 50 - `10.10.50.0/24` | HR | `10.10.50.1` | PC3 on Switch1; PC9 on Switch3 |
| `10.10.20.0/24` | Guest | `10.10.20.1` | PC4, PC5, PC6 on Switch2 |

Switch1 and Switch3 are connected by an **802.1Q trunk** carrying VLANs 10, 40, and 50.

The old R1 router-on-a-stick subinterfaces `eth0.10`, `eth0.40`, and `eth0.50` are no longer used. Inter-VLAN routing and policy enforcement for these VLANs now occur on **pfSense-Internal**.

The previous R3-side VLAN 60 and VLAN 70 gateway design is no longer part of the current topology. Guest clients are placed behind **pfSense-Guest** on the `10.10.20.0/24` Guest network.

## 5. Current Client Addressing

Client addresses are assigned dynamically by DHCP. The addresses below are examples observed during current testing and may change when leases are renewed.

| Client | Network | Observed / Assignment |
|---|---|---|
| PC1 | IT / VLAN 10 | DHCP; observed `10.10.10.100/24` |
| PC2 | Finance / VLAN 40 | DHCP; observed `10.10.40.100/24` |
| PC3 | HR / VLAN 50 | DHCP; observed `10.10.50.100/24` |
| PC4 | Guest | DHCP on `10.10.20.0/24`; `10.10.20.100` used during testing |
| PC5 | Guest | DHCP on `10.10.20.0/24` |
| PC6 | Guest | DHCP on `10.10.20.0/24` |
| PC7 | IT / VLAN 10 | DHCP; observed `10.10.10.101/24` |
| PC8 | Finance / VLAN 40 | DHCP; observed `10.10.40.101/24` |
| PC9 | HR / VLAN 50 | DHCP; observed `10.10.50.101/24` |
| webterm-IT | IT / VLAN 10 | DHCP; observed `10.10.10.102/24` |

All current client DHCP scopes distribute:

```text
DNS server: 10.10.30.10
```

## 6. DHCP Configuration

Centralized DHCP runs on **DHCP01 (`10.10.30.10`)** using Kea DHCPv4.

| Network | Pool Range | Default Gateway | DNS Server | Relay |
|---|---|---|---|---|
| IT - `10.10.10.0/24` | `10.10.10.100 - 10.10.10.199` | `10.10.10.1` | `10.10.30.10` | pfSense-Internal |
| Guest - `10.10.20.0/24` | `10.10.20.100 - 10.10.20.199` | `10.10.20.1` | `10.10.30.10` | pfSense-Guest |
| Finance - `10.10.40.0/24` | `10.10.40.100 - 10.10.40.199` | `10.10.40.1` | `10.10.30.10` | pfSense-Internal |
| HR - `10.10.50.0/24` | `10.10.50.100 - 10.10.50.199` | `10.10.50.1` | `10.10.30.10` | pfSense-Internal |

For pfSense-Internal, the DHCP relay downstream interfaces are:

- IT
- FINANCE
- HR

The upstream DHCP server is:

```text
10.10.30.10
```

For pfSense-Guest, DHCP requests from the Guest LAN are relayed to the same centralized DHCP server.

## 7. DNS Configuration

BIND9 runs on **DHCP01 (`10.10.30.10`)**.

Clients use:

```text
DNS server: 10.10.30.10
```

BIND9 provides recursive forwarding for external names and authoritative DNS for the internal `gns3.lab` zone.

## 8. Routing Notes

### R1 to pfSense-Internal

R1 no longer owns the IT, Finance, or HR gateway addresses. It uses static routes through pfSense-Internal:

```text
10.10.10.0/24 -> 10.10.252.2
10.10.40.0/24 -> 10.10.252.2
10.10.50.0/24 -> 10.10.252.2
```

These routes are selectively redistributed into OSPF using the `INTERNAL-VLANS` prefix-list and `REDIST-INTERNAL` route-map.

### R3 to pfSense-Guest

R3 reaches the Guest network using:

```text
10.10.20.0/24 -> 10.10.253.2
```

The Guest route is selectively redistributed into OSPF using the `GUEST-VLAN` prefix-list and `REDIST-GUEST` route-map.

### Internet / Default Route

R2 uses FortiGate as its upstream next hop:

```text
0.0.0.0/0 -> 10.10.254.1
```

R2 originates the default route into OSPF so R1 and R3 learn Internet reachability dynamically.

FortiGate provides the perimeter path to the GNS3 NAT node and the external Internet.

## 9. Firewall Gateway Ownership

Current default-gateway ownership is:

```text
IT       10.10.10.1 -> pfSense-Internal
Guest    10.10.20.1 -> pfSense-Guest
Finance  10.10.40.1 -> pfSense-Internal
HR       10.10.50.1 -> pfSense-Internal
Server   10.10.30.1 -> R2
```

This is an important change from the earlier router-on-a-stick design. R1 and R3 remain part of the routed OSPF core, while the pfSense firewalls now provide the Layer 3 gateways and security-policy enforcement for the protected client networks.

## 10. pfSense-Internal Management Access

The former pfSense-Internal LAN/native management interface `192.168.1.1/24` has been removed because it is no longer used.

Management access is now performed from the IT VLAN through the pfSense-Internal IT interface:

```text
pfSense-Internal management address: 10.10.10.1
Protocol: HTTPS (TCP/443)
Allowed source: IT subnet (10.10.10.0/24)
```

The IT firewall policy includes an explicit rule allowing IT clients to reach **This Firewall (self)** over TCP/443. This rule is placed above the rule that blocks other private/internal networks.

Finance, HR, and Guest networks are not granted equivalent pfSense-Internal GUI management access.

`webterm-IT` now operates as a normal IT workstation and receives its address through DHCP instead of using the retired `192.168.1.0/24` management network.


---

## 11. FortiGate Management Access

FortiGate is managed from the trusted IT network using its internal-facing `port2` address:

```text
FortiGate management address: 10.10.254.1
Protocol: HTTPS (TCP/443)
Allowed source: IT subnet (10.10.10.0/24)
```

The management traffic reaches FortiGate through the routed core:

```text
IT VLAN -> pfSense-Internal -> R1 -> R2 -> FortiGate port2
```

The FortiGate interface itself is configured to allow HTTPS management on `port2`. pfSense-Internal has a corresponding IT-interface exception above the private-network block so the trusted IT subnet can reach `10.10.254.1:443`.
