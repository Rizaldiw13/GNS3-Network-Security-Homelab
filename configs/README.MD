# Configuration Files

This directory contains sanitized, human-readable configuration snapshots for the routing and infrastructure services used in the GNS3 Network Security Homelab.

The purpose of this folder is to keep the important lab configuration version-controlled, readable, and easy to reference during troubleshooting or future rebuilds.

## VyOS Routers

- `R1-config.txt` — current R1 configuration, including the pfSense-Internal transit, static routes to IT/Finance/HR, and selective OSPF redistribution.
- `R2-config.txt` — current R2 configuration, including the FortiGate transit, server network, OSPF, and default route toward FortiGate.
- `R3-config.txt` — current R3 configuration, including the pfSense-Guest transit, Guest static route, and selective OSPF redistribution.

The router snapshots are generated from:

```text
show configuration commands
```

Authentication secrets are redacted before committing.

## FortiGate

- `FortiGate-config.txt` — sanitized snapshot of the perimeter interfaces, static return route, and active internal-to-Internet firewall policy.

The FortiGate snapshot intentionally includes only configuration relevant to this lab. Default/inactive interfaces, unique identifiers, and unrelated system settings are omitted.

The `INTERNAL-NETWORKS` address object represents `10.10.0.0/16`.

The active perimeter policy currently permits the `INTERNAL-NETWORKS` source object to use:

- HTTPS
- DNS
- NTP
- PING

from `port2` toward `port1`, with NAT enabled.

The FortiGate has a static return route for `10.10.0.0/16` via R2 at `10.10.254.2`. Its Internet-facing `port1` receives addressing dynamically through DHCP from the GNS3 NAT side.

## DHCP

- `kea-dhcp4.conf.txt` — current Kea DHCPv4 configuration for the active IT, Guest, Finance, and HR networks.

Current scopes:

- `10.10.10.0/24` — IT
- `10.10.20.0/24` — Guest
- `10.10.40.0/24` — Finance
- `10.10.50.0/24` — HR

## DNS

- `named.conf.local` — BIND9 local zone declaration.
- `named.conf.options.txt` — BIND9 recursion, forwarding, query-access, and listen settings.
- `db.gns3.lab.txt` — forward zone file for `gns3.lab`.

The live BIND9 configuration has been validated with:

```text
named-checkconf
named-checkzone gns3.lab /etc/bind/db.gns3.lab
```

## pfSense

Raw pfSense XML backups are intentionally **not published** in this public repository.

The pfSense-Internal and pfSense-Guest configurations are documented through:

- `../docs/firewall-policy.md`
- `../docs/addressing-plan.md`
- `../docs/design-decisions.md`
- `../topology/README.md`

This keeps the public repository focused on readable architecture and policy documentation while avoiding publication of full firewall backups that may contain sensitive configuration data.
