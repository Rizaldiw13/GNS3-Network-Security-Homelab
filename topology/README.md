## Static Routing Topology

These static routes will be replaced by my OSPF implementation. However, I thought it would be neat to show how the topology looks currently

![Static Routing Topology](Starting-Topology.png)

## OSPF Routing

The topology currently uses OSPFv2 for dynamic IPv4 routing.

All routers participate in backbone Area 0.

| Router | Router ID | OSPF Interfaces |
|---|---|---|
| R1 | `1.1.1.1` | eth0, eth1 |
| R2 | `2.2.2.2` | eth0, eth1 |
| R3 | `3.3.3.3` | eth0, eth1 |

### Routing Tables

R1 routing table:
![R1 ROUTING TABLE](R1-OSPF-ROUTING-TABLE.png)
R2 routing table:
![R2 ROUTING TABLE](R2-OSPF-ROUTING-TABLE.png)
R3 routing table:
![R3 ROUTING TABLE](R3-OSPF-ROUTING-TABLE.png)

### OSPF Adjacencies

```text
                          Area 0

       1.1.1.1            2.2.2.2          3.3.3.3
          R1 -------------- R2 -------------- R3
                Full                Full
```