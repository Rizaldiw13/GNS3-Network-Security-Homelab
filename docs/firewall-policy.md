# Firewall Policy

**Updated:** 2026-11-21

This document records the active firewall policy for the current GNS3 Network Security Homelab.

The lab uses three firewall roles:

- **FortiGate** — perimeter firewall between the routed core and the GNS3 NAT/Internet.
- **pfSense-Internal** — segmentation firewall and default gateway for IT, Finance, and HR.
- **pfSense-Guest** — segmentation firewall and default gateway for the Guest network.

The policy follows a **default-deny** approach: traffic is permitted only when an explicit rule allows it, with broader allow rules placed below more specific exceptions and private-network blocks.

---

## 1. pfSense-Internal

### Interfaces

| Interface | Address | Role |
|---|---|---|
| WAN | `10.10.252.2/30` | Transit toward R1 |
| IT | `10.10.10.1/24` | IT gateway / VLAN 10 |
| FINANCE | `10.10.40.1/24` | Finance gateway / VLAN 40 |
| HR | `10.10.50.1/24` | HR gateway / VLAN 50 |

The former LAN interface `192.168.1.1/24` has been removed.

### IT Interface Policy

Rules are evaluated top-down.

| Order | Action | Protocol | Source | Destination | Port | Purpose |
|---:|---|---|---|---|---|---|
| 1 | Allow | UDP | Any, source port 68 | `255.255.255.255` | 67 | Allow DHCP client broadcasts |
| 2 | Allow | TCP/UDP | IT subnets | `10.10.30.10` | 53 | Allow DNS to DNS01 |
| 3 | Allow | IPv4 any | IT subnets | Finance subnets | Any | Allow IT to manage/reach Finance |
| 4 | Allow | IPv4 any | IT subnets | HR subnets | Any | Allow IT to manage/reach HR |
| 5 | Allow | IPv4 any | IT subnets | `10.10.20.0/24` | Any | Allow IT to reach Guest for support |
| 6 | Allow | TCP | IT subnets | This Firewall (self) | 443 | Allow IT clients to manage pfSense-Internal over HTTPS |
| 7 | Block | IPv4 any | IT subnets | `PRIVATE_NETWORKS` | Any | Block other private/internal networks not explicitly allowed above |
| 8 | Allow | IPv4 any | IT subnets | Any | Any | Allow Internet/non-private destinations |

### Finance Interface Policy

| Order | Action | Protocol | Source | Destination | Port | Purpose |
|---:|---|---|---|---|---|---|
| 1 | Allow | UDP | Any, source port 68 | `255.255.255.255` | 67 | Allow DHCP client broadcasts |
| 2 | Allow | TCP/UDP | Finance subnets | `10.10.30.10` | 53 | Allow DNS to DNS01 |
| 3 | Allow | ICMP echo request | Finance subnets | This Firewall (self) | Any | Allow gateway reachability testing |
| 4 | Block | IPv4 any | Finance subnets | `PRIVATE_NETWORKS` | Any | Block access to private/internal networks |
| 5 | Allow | IPv4 any | Finance subnets | Any | Any | Allow Internet/non-private destinations |

Because the private-network block occurs before the final outbound allow, Finance cannot initiate connections to IT, HR, Guest, or other RFC1918 internal networks.

### HR Interface Policy

| Order | Action | Protocol | Source | Destination | Port | Purpose |
|---:|---|---|---|---|---|---|
| 1 | Allow | UDP | Any, source port 68 | `255.255.255.255` | 67 | Allow DHCP client broadcasts |
| 2 | Allow | TCP/UDP | HR subnets | `10.10.30.10` | 53 | Allow DNS to DNS01 |
| 3 | Allow | ICMP echo request | HR subnets | This Firewall (self) | Any | Allow gateway reachability testing |
| 4 | Block | IPv4 any | HR subnets | `PRIVATE_NETWORKS` | Any | Block access to private/internal networks |
| 5 | Allow | IPv4 any | HR subnets | Any | Any | Allow Internet/non-private destinations |

Because the private-network block occurs before the final outbound allow, HR cannot initiate connections to IT, Finance, Guest, or other RFC1918 internal networks.

### WAN Interface Policy

No explicit pass rules are currently configured on the pfSense-Internal WAN interface.

This is intentional.

The WAN interface is the transit link toward R1, not the public Internet. Traffic initiated from IT, Finance, or HR is evaluated when it enters pfSense on the corresponding internal interface. When an allow rule creates a firewall state, related return traffic arriving through WAN is automatically permitted.

Unsolicited traffic arriving from R1/core networks does not match an allow rule and is denied by pfSense's default-deny behavior.

This provides a useful security boundary:

```text
IT / Finance / HR initiated traffic
        -> evaluated on internal interface
        -> allowed if policy permits
        -> state created
        -> reply traffic allowed automatically

Core / remote network initiated traffic
        -> enters pfSense-Internal on WAN
        -> no WAN pass rule
        -> denied by default
```

Do **not** enable a blanket "Block private networks" rule on this WAN interface. The WAN is an internal RFC1918 transit network, and legitimate routed traffic from the enterprise core uses private addressing.

If cross-site administration is implemented later, narrowly scoped WAN rules should be added only for the required trusted sources and management protocols.

Example future rule:

```text
Source: Branch IT subnet
Destination: This Firewall (self)
Protocol: TCP
Port: 443
Action: Allow
```

---

## 2. pfSense-Guest

### Interfaces

| Interface | Address | Role |
|---|---|---|
| WAN | `10.10.253.2/30` | Transit toward R3 |
| LAN | `10.10.20.1/24` | Guest default gateway |

### Guest WAN Policy

| Order | Action | Protocol | Source | Destination | Purpose |
|---:|---|---|---|---|---|
| 1 | Block | Any | Reserved/bogon sources | Any | Block invalid/unallocated source networks |
| 2 | Allow | TCP | `10.10.10.0/24` | WAN address (`10.10.253.2`) | 443 | Allow HQ IT to manage pfSense-Guest over HTTPS |
| 3 | Allow | IPv4 any | `10.10.10.0/24` | LAN subnets | Any | Allow HQ/IT access to Guest hosts for support |
| 4 | Allow | ICMP echo request | `10.10.253.1` | `10.10.253.2` | Any | Allow R3 to ping the pfSense-Guest WAN interface |

The HTTPS management rule allows trusted IT clients from `10.10.10.0/24` to reach the pfSense-Guest web interface at `https://10.10.253.2`.

The following rule separately permits IT clients from `10.10.10.0/24` to initiate connections toward Guest hosts for support and troubleshooting.

### Guest LAN Policy

| Order | Action | Protocol | Source | Destination | Port | Purpose |
|---:|---|---|---|---|---|---|
| Auto | Allow | TCP | LAN clients | LAN Address | 443/80 | pfSense anti-lockout rule |
| 1 | Allow | TCP/UDP | LAN subnets | `10.10.30.10` | 53 | Allow DNS to DNS01 |
| 2 | Allow | ICMP echo request | LAN subnets | This Firewall (self) | Any | Allow gateway reachability testing |
| 3 | Block | IPv4 any | LAN subnets | `PRIVATE_NETWORKS` | Any | Block Guest access to internal/private networks |
| 4 | Allow | IPv4 any | LAN subnets | Any | Any | Allow Internet/non-private destinations |

The DNS exception must be above the private-network block because DNS01 (`10.10.30.10`) is itself inside private RFC1918 address space.

### Guest Management Hardening

Trusted remote management from the HQ IT network is now explicitly configured on the pfSense-Guest WAN interface:

```text
Source:      10.10.10.0/24
Destination: WAN address (10.10.253.2)
Protocol:    TCP
Port:        443
Action:      Allow
```

This gives HQ IT a dedicated management path to the pfSense-Guest GUI without requiring broad access from the Guest LAN.

The default pfSense anti-lockout rule on the Guest LAN should only be disabled after the HQ IT management path has been tested successfully. Until that anti-lockout rule is removed, Guest-LAN clients may still be able to reach the firewall web interface on ports 80/443.

---

## 3. Effective Segmentation Matrix

| Source | IT | Finance | HR | Guest | DNS01 | Internet | pfSense-Internal GUI | pfSense-Guest GUI |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| IT | Local | Allow | Allow | Allow | DNS allowed | Allow | HTTPS allowed | HTTPS allowed |
| Finance | Block | Local | Block | Block | DNS allowed | Allow | Block | Block |
| HR | Block | Block | Local | Block | DNS allowed | Allow | Block | Block |
| Guest | Block | Block | Block | Local | DNS allowed | Allow | Not intended | Anti-lockout remains until removed |
| Core/other networks | Default deny at pfSense-Internal WAN | Default deny | Default deny | Depends on Guest WAN rules | N/A | N/A | Default deny | Default deny except explicit WAN rules |

---

## 4. Stateful Firewall Behavior

pfSense is stateful.

When a permitted connection is initiated, pfSense creates a state entry containing information such as source, destination, protocol, and ports.

This means a reverse-direction rule is not required merely for reply traffic.

Example:

```text
IT client -> Finance host
        allowed by IT interface rule
        state created

Finance host -> reply to same IT session
        allowed by existing state
```

This does **not** mean Finance is allowed to initiate a new connection toward IT. A new Finance-initiated session is evaluated against the Finance interface rules and is blocked by the `PRIVATE_NETWORKS` rule.

---

## 5. Policy Design Principles

The current ruleset follows these principles:

1. **Default deny** — traffic is denied unless explicitly permitted.
2. **Least privilege** — exceptions are added only for required services or administrative access.
3. **Rule ordering matters** — specific exceptions appear above broad private-network blocks.
4. **Centralized infrastructure access** — DHCP and DNS are explicitly allowed before segmentation blocks.
5. **Administrative separation** — IT receives broader access than Finance, HR, and Guest.
6. **Stateful return traffic** — return packets for permitted sessions do not require duplicate reverse rules.
7. **Layered security** — FortiGate provides perimeter protection while pfSense performs internal segmentation.

---

## 6. Current Verified Behavior

The current firewall configuration has been tested successfully for:

- IT Internet access.
- Finance Internet access.
- HR Internet access.
- Guest Internet access.
- IT access to Finance.
- IT access to HR.
- IT access to Guest.
- Finance-to-IT blocking.
- Finance-to-HR blocking.
- HR-to-IT blocking.
- HR-to-Finance blocking.
- Guest-to-private-network blocking.
- DNS access from all client networks to `10.10.30.10`.
- DHCP operation through pfSense relay.
- IT HTTPS management of pfSense-Internal at `10.10.10.1`.
- HQ IT HTTPS management of pfSense-Guest at `10.10.253.2`.

