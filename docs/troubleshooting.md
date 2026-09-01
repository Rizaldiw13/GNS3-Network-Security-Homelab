# Troubleshooting 

**Last updated:** 2026-08-28

This document summarizes the major troubleshooting incidents encountered while building the GNS3 networking homelab. 

---

## 1. GNS3 VM / VMware Hyper-V Conflict

### Symptoms

During the initial GNS3 setup, VMware Workstation repeatedly reported that VMware and Hyper-V were not compatible.

### Investigation

We checked Windows features and settings that can keep the Microsoft hypervisor active, including:

- Memory Integrity / Core Isolation
- Hyper-V-related Windows Features
- Windows Hypervisor Platform
- Virtual Machine Platform
- `bcdedit` hypervisor settings
- Code Integrity policy state

A key command used was:

```text
bcdedit /set hypervisorlaunchtype off
```

### Resolution

Conflicting Hyper-V and virtualization-based security features were disabled so VMware Workstation could run the GNS3 VM reliably.

### Lesson Learned

A GNS3 problem can originate below GNS3 itself. Host virtualization settings should be checked before assuming the topology or GNS3 VM is at fault.

---

## 2. Routing Migration and OSPF Configuration Issues

### Problem

The lab originally relied on static routing. As the topology became more complex, routing was migrated to OSPF.

During this migration, an OSPF configuration-style conflict occurred on R2.

### Error

The following command was attempted:

```text
set protocols ospf area 0 network '10.10.30.0/24'
```

VyOS rejected the commit:

```text
Cannot use OSPF "interface area" and "area network" configuration at the same time!
```

### Root Cause

The routers were already using interface-based OSPF configuration, for example:

```text
set protocols ospf interface eth0 area 0
set protocols ospf interface eth1 area 0
```

VyOS did not allow that style to be mixed with `area network` configuration.

### Resolution

The conflicting command was removed:

```text
delete protocols ospf area 0 network '10.10.30.0/24'
```

R2 `eth2` was then added using the same interface-based style:

```text
set protocols ospf interface eth2 area 0
```

Because `eth2` connects to the server network rather than another OSPF router, it was also made passive:

```text
set protocols ospf interface eth2 passive
```

### Why R2 `eth2` Is Passive

R2 still needs to advertise:

```text
10.10.30.0/24
```

into OSPF so R1 and R3 can reach DHCP01.

However, DHCP01 is an Ubuntu server and does not run OSPF. Therefore, R2 should not send OSPF Hello packets or try to establish an OSPF adjacency on that interface.

```text
Advertise 10.10.30.0/24 into OSPF: Yes
Send OSPF Hellos toward DHCP01:    No
Form OSPF adjacency on eth2:       No
```

### Lesson Learned

When extending an existing routing configuration, use the same configuration style already in use. Passive interfaces are useful for advertising LAN or server networks without forming unnecessary routing adjacencies.

---

## 3. OSPF Preferred Path and Failover

### Problem

After adding the direct R1-R3 transit link, we needed to verify that OSPF preferred it and could fail over automatically if the link went down.

Transit networks:

```text
R1-R2: 10.10.100.0/30
R2-R3: 10.10.100.4/30
R1-R3: 10.10.100.8/30
```

### Normal Operation

R1 learned LAN2 through the direct R1-R3 link:

```text
10.10.20.0/24 via 10.10.100.10, eth2
```

Normal path:

```text
PC1 -> R1 -> R3 -> PC2
```

OSPF cost:

```text
2
```

### Failure Simulation

R1 `eth2` was temporarily disabled:

```text
set interfaces ethernet eth2 disable
```

The direct R1-R3 adjacency disappeared and R1 learned LAN2 through R2:

```text
10.10.20.0/24 via 10.10.100.2, eth1
```

Backup path:

```text
PC1 -> R1 -> R2 -> R3 -> PC2
```

The OSPF cost increased to:

```text
3
```

### Verification

During the failure:

- PC1 could still ping PC2.
- PC2 could still ping PC1.
- OSPF automatically recalculated the route.
- No manual backup route was required.

The link was restored with:

```text
delete interfaces ethernet eth2 disable
```

### Lesson Learned

OSPF automatically selects and recalculates paths. Routing tables and neighbor state are the best evidence that convergence happened as expected.

---

## 4. Original DHCP Design on R1 and PC1 DHCP Failure

### Original Design

R1 originally acted as the centralized DHCP server for both LANs.

The pools were:

```text
LAN1: 10.10.10.100 to 10.10.10.199
LAN2: 10.10.20.100 to 10.10.20.199
```

LAN2 DHCP requests were relayed through R3.

### Relay Troubleshooting

While configuring the original relay design, I was debating on which R1 address should be used as the DHCP server/listen address and why transit addresses mattered.

Addresses tested during troubleshooting included:

```text
10.10.10.1
10.10.100.1
10.10.100.9
```

This clarified that relayed DHCP traffic must reach an address where the server is actually listening and from which it can reply correctly to the relay.

### PC1 Failure

PC1 later failed to obtain an address directly from R1:

```text
DDD
Can't find dhcp server
```

However, when PC1 was given:

```text
10.10.10.10/24
Gateway: 10.10.10.1
```

it could ping R1 successfully.

That proved the basic LAN connection was working.

### Packet Capture

Wireshark showed DHCP Discover broadcasts leaving PC1:

```text
0.0.0.0 -> 255.255.255.255
DHCP Discover
```

This confirmed the client was transmitting correctly.

### Architectural Decision

Instead of continuing to make R1 serve both as a router and centralized DHCP server, the lab was redesigned around a dedicated DHCP server.

### Lesson Learned

Packet capture is useful for separating client-side failures from server or relay failures. Seeing the Discover on the wire proved that PC1 was doing its part correctly.

---

## 5. Deploying DHCP01 and the Server Network

### Goal

A dedicated Ubuntu Server VM named `DHCP01` was added to host Kea DHCPv4.

New server network:

```text
10.10.30.0/24
```

Addressing:

```text
R2 eth2:     10.10.30.1/24
DHCP01 ens3: 10.10.30.10/24
```

### Initial Problem

After Ubuntu installation, `ens3` was up but did not have the required IPv4 address.

### Resolution

Netplan was configured with:

```text
10.10.30.10/24
```

and a default route through:

```text
10.10.30.1
```

`sudo netplan try` was used to safely test the configuration.

### Verification

DHCP01 successfully pinged R2 and R2 successfully pinged DHCP01.

### Lesson Learned

Infrastructure servers should use predictable static addresses. `netplan try` is useful because a bad network change can be automatically reverted.

---

## 6. Internet Access: Routing, NAT, and DNS

This issue involved several separate layers.

### Stage 1: DHCP01 Could Reach R2 but Not the Internet

DHCP01 could ping:

```text
10.10.30.1
```

but could not ping:

```text
8.8.8.8
```

The response was:

```text
From 10.10.30.1 Destination Net Unreachable
```

### Root Cause

R2 did not yet have an upstream Internet path.

### Resolution: Add GNS3 NAT

A GNS3 NAT node was connected to R2 `eth3`.

R2 `eth3` was configured as a DHCP client:

```text
set interfaces ethernet eth3 address 'dhcp'
```

Observed during testing:

```text
R2 eth3:     192.168.42.234/24
NAT gateway: 192.168.42.1
```

R2 also received:

```text
0.0.0.0/0 via 192.168.42.1
```

R2 could then ping `8.8.8.8`.

### Stage 2: R2 Had Internet but DHCP01 Still Did Not

DHCP01 still could not reach the Internet.

### Root Cause

Traffic leaving DHCP01 still used its private source address:

```text
10.10.30.10
```

The upstream network did not have a return route for the internal `10.10.0.0/16` networks.

### Resolution: Source NAT Masquerading

R2 was configured to masquerade internal traffic leaving `eth3`.

Conceptually:

```text
10.10.30.10
    |
    | source NAT
    v
192.168.42.234
```

After source NAT was enabled, DHCP01 could ping `8.8.8.8`.

### Stage 3: Internet Worked by IP but DNS Failed

DHCP01 could reach public IPs, but:

```text
ping archive.ubuntu.com
```

failed with:

```text
Temporary failure in name resolution
```

### Resolution

DNS resolvers were added to DHCP01's Netplan configuration:

```text
8.8.8.8
1.1.1.1
```

After applying Netplan, hostname resolution worked.

### Lesson Learned

Internet access is made up of separate functions:

```text
Routing -> tells packets where to go
NAT     -> makes private traffic returnable
DNS     -> translates names into IP addresses
```

---

## 7. Kea DHCP Installation, Permissions, and Configuration

### Service Discovery Confusion

The `kea-dhcp4-server` package was installed, but the service initially appeared unavailable.

Systemd checks later confirmed the service:

```text
kea-dhcp4-server.service
```

and showed:

```text
Active: active (running)
```

### Kea Directory Permissions

The normal `labadmin` user could not list:

```text
/etc/kea/
```

The directory belonged to:

```text
_kea:_kea
```

The systemd unit showed:

```text
User=_kea
```

### Validation as the Service User

The configuration was validated as the same account used by the service:

```text
sudo -u _kea /usr/sbin/kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

### Syntax Errors Fixed

Several configuration mistakes were caught:

- Duplicate subnet ID
- Missing quotation mark around `10.10.20.0/24`
- Missing comma after `"id": 2`

The LAN2 block was corrected to use:

```json
{
  "id": 2,
  "subnet": "10.10.20.0/24"
}
```

### Interface Listening Problem

The original Kea logs showed:

```text
DHCPSRV_NO_SOCKETS_OPEN
```

because no interface had been configured for DHCP traffic.

The config was updated with:

```json
"interfaces-config": {
  "interfaces": ["ens3"]
}
```

### Final Result

The configuration validated successfully and Kea restarted as:

```text
Active: active (running)
```

### Final Kea DHCP Configuration
The final `/etc/kea/kea-dhcp4.conf` used for the deployment was:

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": ["ens3"]
    },

    "lease-database": {
      "type": "memfile",
      "persist": true
    },

    "valid-lifetime": 86400,
    "renew-timer": 43200,
    "rebind-timer": 75600,

    "subnet4": [
      {
        "id": 1,
        "subnet": "10.10.10.0/24",
        "pools": [
          {
            "pool": "10.10.10.100 - 10.10.10.199"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "10.10.10.1"
          }
        ]
      },

      {
        "id": 2,
        "subnet": "10.10.20.0/24",
        "pools": [
          {
            "pool": "10.10.20.100 - 10.10.20.199"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "10.10.20.1"
          }
        ]
      }
    ]
  }
}
```

Key parts of the configuration:

- `interfaces-config` tells Kea to listen on DHCP01's `ens3` interface.
- `lease-database` uses Kea's `memfile` backend and persists lease information to disk.
- `valid-lifetime` is `86400` seconds, or 1 day.
- `renew-timer` is `43200` seconds, or 12 hours.
- `rebind-timer` is `75600` seconds, or 21 hours.
- Subnet ID `1` represents LAN1 (`10.10.10.0/24`).
- Subnet ID `2` represents LAN2 (`10.10.20.0/24`).
- Each subnet has its own DHCP pool and correct default gateway.
- DNS is intentionally not configured yet for the client DHCP scopes.

The configuration was validated before restarting Kea with:

```text
sudo -u _kea /usr/sbin/kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

### Lesson Learned

Validate configuration before restarting a service. Running checks as the actual service user can also expose permission issues that are hidden when testing as root.

---

## 8. Migrating to Centralized DHCP with Relay Agents

### Final Architecture

DHCP service was removed from R1.

R1 became the relay for LAN1:

```text
listen-interface:   eth0
server:             10.10.30.10
upstream-interface: eth1
upstream-interface: eth2
```

R3's relay destination was changed from:

```text
10.10.10.1
```

to:

```text
10.10.30.10
```

R3 became the relay for LAN2:

```text
listen-interface:   eth1
server:             10.10.30.10
upstream-interface: eth0
upstream-interface: eth2
```

### Verification

PC1 received:

```text
IP:          10.10.10.100/24
Gateway:     10.10.10.1
DHCP Server: 10.10.30.10
```

PC2 received:

```text
IP:          10.10.20.100/24
Gateway:     10.10.20.1
DHCP Server: 10.10.30.10
```

Kea logs confirmed the DORA process:

```text
DHCPDISCOVER
DHCPOFFER
DHCPREQUEST
DHCPACK
```

### Lesson Learned

DHCP relay allows multiple routed LANs to use one centralized DHCP server without placing a DHCP server on every subnet.

---

## 9. Clients Had DHCP but No Internet Access

### Symptoms

After DHCP was working, PC1 and PC2 initially could not reach `8.8.8.8`.

PC1 received:

```text
10.10.10.1 ... Destination network unreachable
```

### Root Cause

R2 had an Internet default route from the GNS3 NAT node:

```text
0.0.0.0/0 via 192.168.42.1
```

but R1 and R3 did not know that R2 was the Internet exit.

### Resolution

R2 was configured to originate its default route into OSPF:

```text
set protocols ospf default-information originate
```

R1 and R3 then learned:

```text
0.0.0.0/0
```

through OSPF toward R2.

### Verification

Afterward:

```text
PC1 -> 8.8.8.8  successful
PC2 -> 8.8.8.8  successful
```

### Lesson Learned

A router having Internet access does not automatically mean all internal routers know how to reach the Internet.

Both of these were required:

```text
OSPF default route -> gets client traffic to R2
Source NAT         -> makes the traffic returnable
```

---

## 10. Full Redundancy Validation After the DHCP Redesign

### Goal

The original failover test was repeated after centralized DHCP and Internet access had been added.

### Test

The direct R1-R3 link was disabled.

OSPF reconverged from:

```text
R1 -> R3
```

to:

```text
R1 -> R2 -> R3
```

### Results

While the direct link was down:

- PC1 could ping PC2.
- PC2 could ping PC1.
- PC1 could obtain DHCP from `10.10.30.10`.
- PC2 could obtain DHCP from `10.10.30.10`.
- PC1 could ping `8.8.8.8`.
- PC2 could ping `8.8.8.8`.

The link was restored and OSPF returned to the lower-cost direct path.

### Lesson Learned

The final topology does not depend on the direct R1-R3 link for service availability. OSPF routing, centralized DHCP, and Internet connectivity all remained functional through the backup path.

---

## 11. Switch1-to-Switch3 Same-VLAN Traffic Failed

### Problem
PC1/PC7, PC2/PC8, and PC3/PC9 could not communicate even though each pair was configured for the same VLAN and subnet.

### Symptoms
- PC3 could ping its R1 gateway `10.10.50.1`.
- PC3 could not ping PC9 `10.10.50.20`.
- Similar failures occurred for VLAN 10 and VLAN 40 pairs.

### Root Cause
Only one side of the Switch1-Switch3 link had been configured as `dot1q`.

Switch3 `eth4` was a trunk, but Switch1 `eth4` was still an access port.

### Resolution
Both ends of the inter-switch link were configured as 802.1Q trunks:

```text
Switch1 eth4 <-> Switch3 eth4
Type: dot1q
```

### Result
Same-VLAN hosts successfully communicated across the two switches:

```text
PC1 <-> PC7  VLAN 10
PC2 <-> PC8  VLAN 40
PC3 <-> PC9  VLAN 50
```

### Lesson Learned
An 802.1Q trunk must be configured consistently on both sides of a switch-to-switch link.

---

## 12. New VLAN Subnets Did Not Appear in Remote OSPF Routing Tables

### Problem
After adding router-on-a-stick VLAN interfaces to R1 and R3, the new VLAN networks did not initially appear in remote OSPF routing tables.

### Configuration Added

R1:

```text
eth0.10 -> Area 0, passive
eth0.40 -> Area 0, passive
eth0.50 -> Area 0, passive
```

R3:

```text
eth1.20 -> Area 0, passive
eth1.60 -> Area 0, passive
eth1.70 -> Area 0, passive
```

### Root Cause
The old parent interfaces were still configured under OSPF:

```text
R1 eth0
R3 eth1
```

After router-on-a-stick conversion, those parent interfaces no longer had Layer 3 addresses.

### Resolution
The stale parent-interface OSPF entries were removed:

R1:

```text
delete protocols ospf interface eth0
```

R3:

```text
delete protocols ospf interface eth1
```

### Result
R1 learned:

```text
10.10.20.0/24
10.10.60.0/24
10.10.70.0/24
```

R3 learned:

```text
10.10.10.0/24
10.10.40.0/24
10.10.50.0/24
```

### Lesson Learned
When converting a routed physical interface into a router-on-a-stick parent interface, clean up stale Layer 3 protocol configuration on the parent.

---

## 13. DHCP Relay Needed to Move From Parent Interfaces to VLAN VIFs

### Problem
The old DHCP relay configuration listened on physical LAN interfaces:

```text
R1: eth0
R3: eth1
```

After router-on-a-stick conversion, those interfaces no longer represented the individual client broadcast domains.

### Resolution

R1 relay listening interfaces were changed to:

```text
eth0.10
eth0.40
eth0.50
```

R3 relay listening interfaces were changed to:

```text
eth1.20
eth1.60
eth1.70
```

The centralized DHCP server remained:

```text
10.10.30.10
```

### Result
DHCP worked on all six VLANs and all nine PCs.

Current observed leases:

```text
PC1  10.10.10.100
PC7  10.10.10.101

PC2  10.10.40.100
PC8  10.10.40.101

PC3  10.10.50.100
PC9  10.10.50.101

PC4  10.10.20.100
PC5  10.10.60.100
PC6  10.10.70.100
```

---

## 14. BIND Logged IPv6 "Network Unreachable" Messages

### Problem
BIND logs showed failed IPv6 attempts when trying to contact DNS infrastructure.

### Observation
The lab is intentionally IPv4-focused and does not provide routed IPv6 connectivity.

### Resolution
BIND was configured for the lab's IPv4 design, including IPv4 forwarding to:

```text
8.8.8.8
1.1.1.1
```

and IPv6 listening was disabled in the BIND options.

### Result
External DNS forwarding worked successfully.

### Lesson Learned
IPv6 lookup warnings are not necessarily evidence of an IPv4 DNS failure when the environment intentionally lacks IPv6 routing.

---

## 22. DNS Forwarding Validation

### Test
The DNS server was queried directly:

```bash
dig @10.10.30.10 google.com
```

### Result
The response showed:

```text
status: NOERROR
SERVER: 10.10.30.10#53
```

and returned an IPv4 address for `google.com`.

### Interpretation
This confirmed:
- BIND was listening on `10.10.30.10`.
- DNS port 53 was reachable.
- BIND accepted recursive queries.
- The configured external forwarders worked.

---

## 15. Internal DNS Zone Validation

### Configuration
An internal authoritative zone was created:

```text
gns3.lab
```

Initial records:

```text
dhcp01.gns3.lab -> 10.10.30.10
r1.gns3.lab     -> 10.10.100.1
r2.gns3.lab     -> 10.10.100.2
r3.gns3.lab     -> 10.10.100.6
```

### Validation
The BIND configuration was checked with:

```bash
sudo named-checkconf
sudo named-checkzone gns3.lab /etc/bind/db.gns3.lab
```

The zone validator returned:

```text
OK
```

Direct queries returned the expected addresses.

PC1 then successfully resolved and pinged:

```text
r1.gns3.lab
dhcp01.gns3.lab
```

### Result
The centralized DNS server now provides both:
- External recursive/forwarded DNS resolution.
- Internal authoritative DNS for infrastructure names.

---

# Troubleshooting Methodology Used

Across these incidents, the most useful troubleshooting sequence was:

1. Check interface state:
   ```text
   show interfaces
   ip addr
   ```

2. Verify directly connected reachability:
   ```text
   ping <default-gateway>
   ```

3. Inspect routing:
   ```text
   show ip route
   ```

4. Check OSPF state:
   ```text
   show ip ospf neighbor
   ```

5. Use packet capture when behavior is unclear:
   - Wireshark
   - DHCP Discover / Offer / Request / ACK analysis

6. Inspect service logs:
   ```text
   journalctl -u kea-dhcp4-server
   ```

7. Validate service configuration before restarting:
   ```text
   sudo -u _kea /usr/sbin/kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
   ```

8. Change one component at a time and retest.

This approach made it easier to separate Layer 2, Layer 3, routing, DHCP, NAT, DNS, and service-level problems instead of treating every connectivity failure as the same issue.
