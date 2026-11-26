# STP, Multicast Snooping, L2 Security, MVRP, MLAG

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26. Which of these a given switch chip offloads: see [switch-chips.md](./switch-chips.md).

Changing `protocol-mode`, `igmp-snooping` or `dhcp-snooping` resets the bridge and briefly drops links.

## STP / RSTP / MSTP

On `/interface/bridge`:
- `protocol-mode=none|stp|rstp|mstp`, default `rstp`. `mstp` requires `vlan-filtering=yes`.
- `priority`: multiples of `4096`, `0x0000`..`0xf000`, default `0x8000` (32768). **7.24+** warns on non-compliant values.
- `port-cost-mode=long|short` (**7.13+**, default `long`): auto `path-cost` from link rate. `long`: 1G=20000, 10G=2000, 25G=800, 100G=200. Virtual, wifi and non-HW bond ports: 20000 (`long`) / 10 (`short`).
- `forward-reserved-addresses=yes|no` (**7.16+**, default `no`, only with `protocol-mode=none`).
- RouterOS does not run PVST+. `01:00:0C:CC:CC:CD` is forwarded as ordinary multicast.

On `/interface/bridge/port`:
- `priority`: steps of `16`, `0x00`..`0xf0`, default `0x80`.
- `edge=auto|yes|no|yes-discover|no-discover` (default `auto`; since 7.14/7.15 dynamic and wireless non-bridge-mode ports become edge automatically).
- `point-to-point=auto|yes|no` (default `auto`; non-wireless ports default to point-to-point since 7.14).
- `bpdu-guard=yes`: a BPDU disables the port until you disable and re-enable it manually.
- `restricted-role=yes`: root guard, keeps the port Alternate while superior BPDUs arrive.
- `restricted-tcn=yes`, `auto-isolate=yes`.

### MSTP

```routeros
/interface/bridge/set bridge1 protocol-mode=mstp region-name=REGION1 region-revision=1
/interface/bridge/msti/add bridge=bridge1 identifier=1 priority=0x1000 vlan-mapping=10,20
```

- `identifier=1..31`. VLANs not mapped to any MSTI belong to MSTI0.
- Per-port, per-MSTI tuning: `/interface/bridge/port/mst-override` (`internal-path-cost`, `priority`).
- All switches in a region must show the same `mst-config-digest` in `/interface/bridge/monitor`.
- MLAG together with MSTP: **7.14+**.

## IGMP / MLD Snooping

```routeros
/interface/bridge/set bridge1 igmp-snooping=yes
```

- Unknown multicast is only restricted after an IGMP/MLD querier is detected, or the bridge runs `multicast-querier=yes` itself.
- MLD snooping needs an active IPv6 address on the bridge.
- `multicast-querier=yes` sends **untagged** queries only. For queriers on several VLANs use IGMP Proxy or PIM-SM on `/interface/vlan`.
- `querier-uses-bridge-address=yes|no` (**7.24+**, documented default `yes`): queries use the bridge IPv4 instead of `0.0.0.0`.
- `last-member-interval` was renamed `last-member-query-interval` in **7.25**.
- CRS3xx with `98DX3236` / `98DX224S` / `98DX226S`: once any querier is seen, the chip drops **all** unknown multicast (including non-IP/IPv6 if only an IGMP querier is seen), except link-local ranges such as `224.0.0.0/24` and `ff02::1`.

Static MDB entry (`ports=` was renamed `interface=` in **7.19**):

```routeros
/interface/bridge/mdb/add bridge=bridge1 group=229.10.10.10 interface=ether2,ether3 vid=10
```

## Bridge L2 Security

Bridge-level switch plus per-port trust flag:

| Feature | Bridge property | Port trust property | Since |
|---|---|---|---|
| DHCP snooping | `dhcp-snooping=yes` | | |
| DHCP option 82 | `dhcp-agent-circuit-id`, `dhcp-agent-remote-id` (replaced `add-dhcp-option82`) | | 7.23 |
| DHCPv6 snooping | `dhcpv6-snooping=yes` | | 7.23 |
| RA guard | `ra-guard=yes` | `trusted-ra` | 7.22 |
| ARP inspection | `arp-inspection=yes` | `trusted-arp` | 7.25 |
| IP source guard | `ip-source-guard=yes` | `trusted-ipsg` | 7.25 |

- DHCP snooping disables bridge FastPath.
- ARP inspection and IP source guard are new in 7.25; detailed operational limits and chip-specific ACL limits are not yet documented. Test before relying on them.

## MVRP (7.15+)

`/interface/bridge` `mvrp=yes`; per port `mvrp-registrar-state`, `mvrp-applicant-state`; `/interface/bridge/vlan` `mvrp-forbidden`; `/interface/vlan` `mvrp=yes`.

## MLAG (7.22+ layout)

Since 7.22, MLAG lives on `/interface/bridge` itself (the `/interface/bridge/mlag` submenu is gone; downgrading loses MLAG config): `mlag-peer-port`, `mlag-priority`, `mlag-heartbeat`, and since 7.25 `mlag-init-delay`, `mlag-lacp-fallback`. HW MLAG is Marvell Prestera only.
