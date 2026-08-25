# Network Addressing Plan

## Overall Address Space

The homelab uses the following RFC1918 private address block:

`10.10.0.0/16`

Smaller subnets are allocated from this address space using Variable Length Subnet Masking (VLSM).

## Current Subnets

| Network | Purpose | Subnet | Mask |
|---|---|---|---|
| Network 1 | PC1 LAN | `10.10.10.0/24` | `255.255.255.0` |
| Network 2 | R1-R2 transit | `10.10.100.0/30` | `255.255.255.252` |
| Network 3 | R2-R3 transit | `10.10.100.4/30` | `255.255.255.252` |
| Network 4 | PC2 LAN | `10.10.20.0/24` | `255.255.255.0` |
| Network 5 | R1-R3 redundant transit | `10.10.100.8/30` | `255.255.255.252` |

## Interface Addressing

| Device | Interface | IP Address | Purpose |
|---|---|---|---|
| PC1 | eth0 | `10.10.10.10/24` | LAN host |
| R1 | eth0 | `10.10.10.1/24` | PC1 default gateway |
| R1 | eth1 | `10.10.100.1/30` | R1-R2 transit |
| R1 | eth2 | `10.10.100.9/30` | R1-R3 redundant transit |
| R2 | eth0 | `10.10.100.2/30` | R1-R2 transit |
| R2 | eth1 | `10.10.100.5/30` | R2-R3 transit |
| R3 | eth0 | `10.10.100.6/30` | R2-R3 transit |
| R3 | eth1 | `10.10.20.1/24` | PC2 default gateway |
| R3 | eth2 | `10.10.100.10/30` | R1-R3 redundant transit |
| PC2 | eth0 | `10.10.20.10/24` | LAN host |

## Transit Address Pool

`10.10.100.0/24` is reserved for router transit networks.

### R1-R2

`10.10.100.0/30`

- Network ID: `10.10.100.0`
- R1 eth1: `10.10.100.1`
- R2 eth0: `10.10.100.2`
- Broadcast: `10.10.100.3`

### R2-R3

`10.10.100.4/30`

- Network ID: `10.10.100.4`
- R2 eth1: `10.10.100.5`
- R3 eth0: `10.10.100.6`
- Broadcast: `10.10.100.7`

### R1-R3 Redundant Link

`10.10.100.8/30`

- Network ID: `10.10.100.8`
- R1 eth2: `10.10.100.9`
- R3 eth2: `10.10.100.10`
- Broadcast: `10.10.100.11`

Future `/30` transit links can continue with:

- `10.10.100.12/30`
- `10.10.100.16/30`
- `10.10.100.20/30`

and so on.
