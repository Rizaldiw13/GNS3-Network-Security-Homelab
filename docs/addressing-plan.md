# Addressing Plan

**Updated:** 2026-09-01

## 1. Overall Address Space

The lab uses **10.10.0.0/16 (RFC1918)** as the overall private IPv4 address space.

## 2. Network Summary

| Network | Subnet Mask | Usable Hosts | Purpose | Gateway / Notes | Address Assignment |
|---|---|---|---|---|---|
| 10.10.10.0 | /24 (255.255.255.0) | 10.10.10.1 to 10.10.10.254 | VLAN 10 - R1-side user network | Gateway: 10.10.10.1 (R1 eth0.10) | DHCP from DHCP01 via R1 relay |
| 10.10.20.0 | /24 (255.255.255.0) | 10.10.20.1 to 10.10.20.254 | VLAN 20 - R3-side user network | Gateway: 10.10.20.1 (R3 eth1.20) | DHCP from DHCP01 via R3 relay |
| 10.10.30.0 | /24 (255.255.255.0) | 10.10.30.1 to 10.10.30.254 | Server network - DHCP01 / DNS | Gateway: 10.10.30.1 (R2 eth2) | Static |
| 10.10.40.0 | /24 (255.255.255.0) | 10.10.40.1 to 10.10.40.254 | VLAN 40 - R1-side user network | Gateway: 10.10.40.1 (R1 eth0.40) | DHCP from DHCP01 via R1 relay |
| 10.10.50.0 | /24 (255.255.255.0) | 10.10.50.1 to 10.10.50.254 | VLAN 50 - R1-side user network | Gateway: 10.10.50.1 (R1 eth0.50) | DHCP from DHCP01 via R1 relay |
| 10.10.60.0 | /24 (255.255.255.0) | 10.10.60.1 to 10.10.60.254 | VLAN 60 - R3-side user network | Gateway: 10.10.60.1 (R3 eth1.60) | DHCP from DHCP01 via R3 relay |
| 10.10.70.0 | /24 (255.255.255.0) | 10.10.70.1 to 10.10.70.254 | VLAN 70 - R3-side user network | Gateway: 10.10.70.1 (R3 eth1.70) | DHCP from DHCP01 via R3 relay |
| 10.10.100.0 | /30 (255.255.255.252) | 10.10.100.1 to 10.10.100.2 | R1-R2 transit | Point-to-point link | Static |
| 10.10.100.4 | /30 (255.255.255.252) | 10.10.100.5 to 10.10.100.6 | R2-R3 transit | Point-to-point link | Static |
| 10.10.100.8 | /30 (255.255.255.252) | 10.10.100.9 to 10.10.100.10 | R1-R3 direct transit | Preferred path | Static |
| 192.168.42.0* | /24 (255.255.255.0) | 192.168.42.1 to 192.168.42.254 | NAT-facing network (GNS3 NAT) | Gateway / NAT IP: 192.168.42.1 | R2 eth3 via GNS3 NAT DHCP |

\* Provided by the GNS3 NAT node. R2 eth3 obtains its outside address dynamically from the NAT node's DHCP service.

## 3. IP Address Assignments

| Device | Interface | IP Address / Prefix | Connected To | Purpose / Notes |
|---|---|---|---|---|
| R1 | eth0 | No IP | Switch1 trunk | Parent interface for router-on-a-stick |
| R1 | eth0.10 | 10.10.10.1/24 | VLAN 10 | Default gateway for VLAN 10; OSPF passive |
| R1 | eth0.40 | 10.10.40.1/24 | VLAN 40 | Default gateway for VLAN 40; OSPF passive |
| R1 | eth0.50 | 10.10.50.1/24 | VLAN 50 | Default gateway for VLAN 50; OSPF passive |
| R1 | eth1 | 10.10.100.1/30 | R2 eth0 | R1-R2 transit |
| R1 | eth2 | 10.10.100.9/30 | R3 eth2 | Direct R1-R3 transit (preferred path) |
| R2 | eth0 | 10.10.100.2/30 | R1 eth1 | R1-R2 transit |
| R2 | eth1 | 10.10.100.5/30 | R3 eth0 | R2-R3 transit |
| R2 | eth2 | 10.10.30.1/24 | DHCP01 | Server network gateway; OSPF passive |
| R2 | eth3 | DHCP (192.168.42.x/24)* | GNS3 NAT | NAT-facing Internet interface |
| R3 | eth0 | 10.10.100.6/30 | R2 eth1 | R2-R3 transit |
| R3 | eth1 | No IP | Switch2 trunk | Parent interface for router-on-a-stick |
| R3 | eth1.20 | 10.10.20.1/24 | VLAN 20 | Default gateway for VLAN 20; OSPF passive |
| R3 | eth1.60 | 10.10.60.1/24 | VLAN 60 | Default gateway for VLAN 60; OSPF passive |
| R3 | eth1.70 | 10.10.70.1/24 | VLAN 70 | Default gateway for VLAN 70; OSPF passive |
| R3 | eth2 | 10.10.100.10/30 | R1 eth2 | Direct R1-R3 transit (preferred path) |
| DHCP01 | ens3 | 10.10.30.10/24 | R2 eth2 | Kea DHCP + BIND9 DNS server |
| DHCP01 | ens8 | Optional / unused | - | Not used |
| PC1 | eth0 | DHCP: 10.10.10.100/24 (current lease) | Switch1 VLAN 10 | DNS 10.10.30.10 |
| PC2 | eth0 | DHCP: 10.10.40.100/24 (current lease) | Switch1 VLAN 40 | DNS 10.10.30.10 |
| PC3 | eth0 | DHCP: 10.10.50.100/24 (current lease) | Switch1 VLAN 50 | DNS 10.10.30.10 |
| PC4 | eth0 | DHCP: 10.10.20.100/24 (current lease) | Switch2 VLAN 20 | DNS 10.10.30.10 |
| PC5 | eth0 | DHCP: 10.10.60.100/24 (current lease) | Switch2 VLAN 60 | DNS 10.10.30.10 |
| PC6 | eth0 | DHCP: 10.10.70.100/24 (current lease) | Switch2 VLAN 70 | DNS 10.10.30.10 |
| PC7 | eth0 | DHCP: 10.10.10.101/24 (current lease) | Switch3 VLAN 10 | DNS 10.10.30.10 |
| PC8 | eth0 | DHCP: 10.10.40.101/24 (current lease) | Switch3 VLAN 40 | DNS 10.10.30.10 |
| PC9 | eth0 | DHCP: 10.10.50.101/24 (current lease) | Switch3 VLAN 50 | DNS 10.10.30.10 |

\* Current observed R2 eth3 address during earlier testing: **192.168.42.234/24**. Because this interface uses DHCP from the GNS3 NAT node, the address is not guaranteed to remain the same.

## 4. VLAN Layout

| VLAN | Subnet | Gateway | Main Clients / Location |
|---|---|---|---|
| 10 | 10.10.10.0/24 | 10.10.10.1 | PC1 on Switch1, PC7 on Switch3 |
| 20 | 10.10.20.0/24 | 10.10.20.1 | PC4 on Switch2 |
| 40 | 10.10.40.0/24 | 10.10.40.1 | PC2 on Switch1, PC8 on Switch3 |
| 50 | 10.10.50.0/24 | 10.10.50.1 | PC3 on Switch1, PC9 on Switch3 |
| 60 | 10.10.60.0/24 | 10.10.60.1 | PC5 on Switch2 |
| 70 | 10.10.70.0/24 | 10.10.70.1 | PC6 on Switch2 |

Switch1 and Switch3 are connected by an **802.1Q trunk**, so VLANs 10, 40, and 50 span both switches at Layer 2.

Switch2 is a separate routed access switch behind R3 and carries VLANs 20, 60, and 70.

## 5. DHCP Configuration

Centralized DHCP runs on **DHCP01 (10.10.30.10)** using Kea DHCPv4.

| VLAN | Scope | Pool Range | Default Gateway | DNS Server |
|---|---|---|---|---|
| 10 | 10.10.10.0/24 | 10.10.10.100 to 10.10.10.199 | 10.10.10.1 | 10.10.30.10 |
| 20 | 10.10.20.0/24 | 10.10.20.100 to 10.10.20.199 | 10.10.20.1 | 10.10.30.10 |
| 40 | 10.10.40.0/24 | 10.10.40.100 to 10.10.40.199 | 10.10.40.1 | 10.10.30.10 |
| 50 | 10.10.50.0/24 | 10.10.50.100 to 10.10.50.199 | 10.10.50.1 | 10.10.30.10 |
| 60 | 10.10.60.0/24 | 10.10.60.100 to 10.10.60.199 | 10.10.60.1 | 10.10.30.10 |
| 70 | 10.10.70.0/24 | 10.10.70.100 to 10.10.70.199 | 10.10.70.1 | 10.10.30.10 |

Lease timers:

- Valid lifetime: 86400 seconds (1 day)
- Renew timer: 43200 seconds (12 hours)
- Rebind timer: 75600 seconds (21 hours)

## 6. DHCP Relay

R1 relays DHCP requests for the R1-side VLANs to DHCP01:

- `eth0.10` for VLAN 10
- `eth0.40` for VLAN 40
- `eth0.50` for VLAN 50
- DHCP server: `10.10.30.10`
- Upstream interfaces: `eth1`, `eth2`

R3 relays DHCP requests for the R3-side VLANs to DHCP01:

- `eth1.20` for VLAN 20
- `eth1.60` for VLAN 60
- `eth1.70` for VLAN 70
- DHCP server: `10.10.30.10`
- Upstream interfaces: `eth0`, `eth2`

## 7. DNS Configuration

BIND9 runs on **DHCP01 (10.10.30.10)**.

All DHCP scopes distribute:

```text
DNS server: 10.10.30.10
```

BIND9 performs two roles:

1. Recursive forwarding for external names through upstream resolvers:
   - `8.8.8.8`
   - `1.1.1.1`
2. Authoritative DNS for the internal `gns3.lab` zone.

Current internal records include:

| Name | Address |
|---|---|
| `dhcp01.gns3.lab` | 10.10.30.10 |
| `r1.gns3.lab` | 10.10.100.1 |
| `r2.gns3.lab` | 10.10.100.2 |
| `r3.gns3.lab` | 10.10.100.6 |

Client PCs are intentionally not assigned permanent manual DNS records because their addresses are dynamically assigned by DHCP.

## 8. Routing, NAT, and Failover Notes

- OSPF Area 0 provides dynamic routing between R1, R2, and R3.
- Client VLAN interfaces on R1 and R3 are configured as OSPF passive interfaces: their subnets are advertised, but no OSPF adjacencies are attempted toward clients.
- R2 eth2 is also OSPF passive while advertising the server network `10.10.30.0/24`.
- R2 originates the default route `0.0.0.0/0` into OSPF.
- R2 performs source NAT masquerading for internal `10.10.0.0/16` traffic out eth3 toward the GNS3 NAT network.
- The direct R1-R3 link `10.10.100.8/30` is the preferred path.
- During failover testing, disabling R1 eth2 caused OSPF to reroute R1-to-R3 VLAN traffic through R2.
- During that failover, cross-VLAN communication, DHCP renewal, DNS reachability, and Internet connectivity remained operational.
