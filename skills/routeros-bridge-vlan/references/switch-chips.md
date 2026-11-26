# Switch Chip Hardware Offload Matrix

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.

Check offload state with `/interface/bridge/port/print` (`H` flag; port `hw=yes` is the default). Only **one** HW-offloaded bridge per switch chip is supported, except CRS1xx/CRS2xx (up to 7).

## Chips that offload `vlan-filtering=yes`

| Chip (examples) | Offloaded | Not offloaded |
|---|---|---|
| **Marvell Prestera** (CRS3xx, CRS4xx, CRS5xx, CRS8xx, CCR2116, CCR2216, RDS2216) | STP/RSTP, MSTP, VLAN filtering, IGMP snooping, DHCPv4/v6 snooping, ARP inspection, IP source guard, RA guard, bonding (`802.3ad`, `balance-xor`, `active-backup` since 7.17), MLAG | `horizon` |
| **88E6393X** (RB5009), **88E6191X** (CCR2004-16G-2S+), **88E6190** (L009) | STP/RSTP, VLAN filtering (`0x8100` only, no `tag-stacking`), IGMP/DHCPv4 snooping (no Q-in-Q), bonding (`802.3ad`, `balance-xor`, `active-backup`, L2 hash only) | MSTP, MLAG, DHCPv6 snooping, RA guard, ARP inspection, IP source guard, `horizon` |
| **QCA8386** (e.g. hAP be³ Media) | STP/RSTP, VLAN filtering, IGMP/DHCPv4/DHCPv6 snooping, RA guard, bonding (`802.3ad`, `balance-xor`, `active-backup`) | MSTP, ARP inspection, IP source guard, MLAG |
| **MT7621** (hEX, hEX S), **MT7531** (hAP ax lite), **EN7523** (hEX Refresh), **RTL8367** (RB4011, RB1100AHx4) | STP/RSTP, VLAN filtering (`0x8100` only, no `tag-stacking`) | IGMP/DHCP snooping, MSTP, bonding, MLAG |

Notes:
- On Prestera, HW bonding ignores the configured `transmit-hash-policy` and always hashes L2+L3+L4.
- On MT7621 / MT7531 / EN7523 / RTL8367, `/interface/ethernet/switch/vlan` does not exist in v7; use bridge VLAN filtering.
- Marvell-98DX3257 (CRS354) cannot VLAN-filter `0x88a8`/`0x9100` on its 1G ports.

## Chips where `vlan-filtering=yes` disables offload

On these, bridge VLAN filtering works but runs on the CPU. For wire speed keep the bridge at `vlan-filtering=no` and configure VLANs in the switch menus.

| Chip (examples) | What to use |
|---|---|
| **CRS1xx / CRS2xx** | `/interface/ethernet/switch/ingress-vlan-translation`, `egress-vlan-tag`, `vlan`, and `/interface/ethernet/switch/set drop-if-invalid-or-src-port-not-member-of-vlan-on-ports=...` |
| **QCA8337, Atheros8327, Atheros8316** (RB3011, hAP ac², hAP ac³, cAP ac, Audience, wAP ac, RB450Gx4) | `/interface/ethernet/switch/vlan` plus `/interface/ethernet/switch/port` (`vlan-mode=secure`, `vlan-header=...`, `default-vlan-id=...`). On QCA8337/Atheros8327 leave `vlan-header=leave-as-is`. `default-vlan-id` is needed for VLAN 1 and reserved MACs |
| **Atheros8227, Atheros7240** (hAP lite, hEX lite) | Only STP/RSTP offloaded; `vlan-filtering=yes` forces CPU |
| **IPQ-PPE** (hAP ax², hAP ax³, cAP ax, Chateau ax), **98PX1012** (CCR2004-1G-12S+2XS) | No bridge HW offload features at all. Keep default `protocol-mode=rstp` or set `hw=no`. `protocol-mode=none` enables a broken HW offload on IPQ-PPE |

## Port isolation on switch chips

`/interface/ethernet/switch/port-isolation/set <port> forwarding-override=<ports-or-list>` is available on all switch chips and keeps HW offload. It is ingress-only, per-port, and limited to one switch chip. On QCA8337/Atheros8327/8316/8227/7240, switch `vlan-mode` overrides port isolation unless ACL rules with `new-dst-ports` are used.
