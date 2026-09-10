# Single-Area OSPF Lab (Area 0) — R1–R4, ASBR Default Route

Configs for a 4-router single-area OSPF topology (R1, R2, R3, R4), an access
switch, and PC1. All point-to-point WAN/LAN links use `/30` masks. R1 is the
ASBR that injects a default route learned from its ISP link into OSPF.

## Topology

```
                203.0.113.0/30                 10.0.12.0/30
ISPR1 ---------------------------- R1 -------------------------- R2
 .2      Gig0/0/0        Gig3/0.1 |Gig0/0            Gig0/0.2|   .1|
 (not configured)                 |                            Fa1/0
                          10.0.13.0/30                        |
                                   |Fa1/0                      |10.0.24.0/30
                                   |.1                         |Fa1/0
                                   |                            |.2
                                  R3 --------------------------- R4
                              Fa1/0.2   Fa2/0    10.0.34.0/30   |Fa2/0.2  Gig0/0(.254)
                                        R3 .1            R4 .2   |
                                                                  192.168.4.0/24
                                                                  |
                                                              SW1 -- PC1 (.1)
```

## IP Addressing Plan

| Link / Network        | Prefix | R1        | R2        | R3        | R4        |
|------------------------|--------|-----------|-----------|-----------|-----------|
| ISP link               | /30    | 203.0.113.1 | –       | –         | –         |
| R1 ↔ R2                | /30    | 10.0.12.1 | 10.0.12.2 | –         | –         |
| R1 ↔ R3                | /30    | 10.0.13.1 | –         | 10.0.13.2 | –         |
| R2 ↔ R4                | /30    | –         | 10.0.24.1 | –         | 10.0.24.2 |
| R3 ↔ R4                | /30    | –         | –         | 10.0.34.1 | 10.0.34.2 |
| R4 LAN                 | /24    | –         | –         | –         | 192.168.4.254 |
| Loopback0               | /32    | 1.1.1.1   | 2.2.2.2   | 3.3.3.3   | 4.4.4.4   |

PC1: `192.168.4.1/24`, gateway `192.168.4.254`.

## Files

```
configs/
  R1.txt   - ASBR, Internet link excluded from OSPF, default-information originate
  R2.txt   - Area 0, transit + loopback
  R3.txt   - Area 0, transit + loopback
  R4.txt   - Area 0, transit + LAN (passive) + loopback
  SW1.txt  - Access switch linking R4 to PC1
  PC1.txt  - End-host addressing notes
```

Paste each file's commands into the corresponding device's CLI (e.g., in
Cisco Packet Tracer or GNS3) starting from privileged EXEC mode.

## Design Notes

- **Router IDs** are set explicitly (`router-id x.x.x.x`) to match each
  router's loopback, since loopbacks are the highest/most stable addresses
  and this avoids ambiguity if the router-id would otherwise be picked from
  a physical interface.
- **R1's Internet-facing interface (Gig3/0) is intentionally left out of any
  `network` statement** under `router ospf 1`, per the requirement not to run
  OSPF on R1's Internet link. A static default route
  (`ip route 0.0.0.0 0.0.0.0 203.0.113.2`) points R1 at the ISP, and
  `default-information originate` advertises that default into the OSPF
  domain, making R1 an ASBR.
- **Passive interfaces**: all loopbacks are passive (no need to form
  neighbors on a stub /32), and R4's LAN-facing Gig0/0 is passive since
  PC1/the switch are not OSPF speakers.
- All transit links use `/30` masks with OSPF wildcard `0.0.0.3`.

## Question 5 — What default route(s) appear on R2, R3, and R4?

After R1 is configured with `default-information originate` under
`router ospf 1`, `show ip route` on **R2, R3, and R4** will each show a
single OSPF **external type-2 (E2)** default route pointing back toward R1,
for example:

```
O*E2  0.0.0.0/0 [110/1] via 10.0.12.1, 00:00:12, GigabitEthernet0/0   (on R2)
O*E2  0.0.0.0/0 [110/1] via 10.0.13.1, 00:00:12, FastEthernet1/0      (on R3)
O*E2  0.0.0.0/0 [110/1] via 10.0.24.1, 00:00:12, FastEthernet1/0      (on R4, via R2)
```

Notes:
- The `*` marks it as the candidate default route used for routing.
- It's type **E2** (not E1) because `default-information originate` uses
  E2 metrics by default — the advertised cost stays constant (1, or
  whatever metric was set) regardless of how many hops away a router is,
  rather than accumulating internal OSPF cost along the path.
- On R4, the route is reachable via R2 or R3 (equal-cost, since both paths
  R1→R2→R4 and R1→R3→R4 are two hops) unless the physical/link costs
  differ — check `show ip ospf interface` for actual interface costs if you
  want a single preferred path.
- Only **one** default route appears per router (not one from each
  neighbor), because all routers are receiving the same LSA (Type-5
  AS-external LSA) originated by R1, not multiple independent defaults.

## Verification Commands

```
show ip interface brief
show ip ospf neighbor
show ip route ospf
show ip route 0.0.0.0
show ip ospf database external
```
