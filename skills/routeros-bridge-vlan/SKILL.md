---
name: routeros-bridge-vlan
description: "RouterOS bridge VLAN filtering, switch-chip hardware offload, STP/RSTP/MSTP, IGMP/MLD snooping, port isolation and L3 hardware offloading on MikroTik. Use when: configuring trunk/access/hybrid ports, a management VLAN, or router-on-a-stick; deciding whether a CRS/RB/hEX/hAP switch chip offloads vlan-filtering; avoiding lockout when enabling vlan-filtering (Safe Mode); tuning spanning tree, IGMP snooping, DHCP snooping, RA guard, MVRP or MLAG; enabling L3HW on CRS3xx/CRS5xx/CCR2116/CCR2216; or when the user mentions /interface/bridge, /interface/bridge/vlan, /interface/bridge/port, pvid, frame-types, ingress-filtering, tagged/untagged, /interface/vlan, /interface/ethernet/switch, hw-offload, l3-hw-offloading, port-isolation or horizon."
---

# RouterOS Bridge & VLAN Filtering

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26. Features newer than 7.12 carry their minimum version inline.

General CLI syntax, `[find]`, REST verbs and package install are covered by `routeros-fundamentals`; this skill covers only L2/L3 switching.

## The Model: One Bridge, VLAN Table, VLAN Interfaces

RouterOS v7 does VLAN switching with **one bridge** that has `vlan-filtering=yes`. Three menus work together:

| Menu | Purpose |
|---|---|
| `/interface/bridge` | The bridge itself, also the **CPU port** (has its own `pvid`, `frame-types`) |
| `/interface/bridge/port` | Per-port `pvid`, `frame-types`, `ingress-filtering`, STP, `hw` |
| `/interface/bridge/vlan` | VLAN membership table: which ports are `tagged` / `untagged` per `vlan-ids` |
| `/interface/vlan` | L3 VLAN interface on the bridge, for router/management IPs |

Key properties:

| Property | Values / default | Meaning |
|---|---|---|
| `vlan-filtering` (bridge) | `yes`/`no`, default `no` | `no` ignores tags (shared VLAN learning). `yes` enables tag processing, per-VLAN learning and port membership |
| `pvid` (bridge and port) | `1..4094`, default `1` | VLAN assigned to untagged ingress frames. No effect until `vlan-filtering=yes` |
| `frame-types` | `admit-all` (default), `admit-only-vlan-tagged`, `admit-only-untagged-and-priority-tagged` | Filters ingress **and** egress |
| `ingress-filtering` | `yes`/`no`, default `yes` in v7 | Drops ingress frames whose VLAN is not in `/interface/bridge/vlan` for that port |
| `ether-type` (bridge) | `0x8100` (default), `0x88a8`, `0x9100` | Which outer tag the bridge inspects (Q-in-Q) |
| `tag-stacking` (port) | `yes`/`no`, default `no` | Treat all ingress as untagged and push an outer `pvid` tag |

**Dynamic membership:** a port with `frame-types=admit-all` or `admit-only-untagged-and-priority-tagged` is **automatically added as an untagged member of its `pvid` VLAN**. You usually do not need `untagged=` entries for access ports. `admit-only-vlan-tagged` prevents this.

Version notes:
- **7.16+**: adding `/interface/vlan` on a `vlan-filtering=yes` bridge auto-creates a dynamic entry tagging the bridge (`comment="added by vlan on bridge"`).
- **7.17+**: `tagged=` / `untagged=` in `/interface/bridge/vlan` accept interface lists.
- **7.20+**: a dynamic tagged `switch-cpu` entry is added when a VLAN spans several switch chips or mixes HW and SW ports.

## Standard Pattern: Trunk + Access + Management VLAN

```routeros
/interface/bridge/add name=bridge1 vlan-filtering=no frame-types=admit-only-vlan-tagged
/interface/bridge/port
add bridge=bridge1 interface=ether1 frame-types=admit-only-vlan-tagged
add bridge=bridge1 interface=ether2 pvid=20 frame-types=admit-only-untagged-and-priority-tagged
add bridge=bridge1 interface=ether3 pvid=30 frame-types=admit-only-untagged-and-priority-tagged
/interface/bridge/vlan
add bridge=bridge1 tagged=ether1 vlan-ids=20
add bridge=bridge1 tagged=ether1 vlan-ids=30
add bridge=bridge1 tagged=bridge1,ether1 vlan-ids=99
/interface/vlan/add interface=bridge1 vlan-id=99 name=MGMT
/ip/address/add address=192.168.99.2/24 interface=MGMT
/interface/bridge/set bridge1 vlan-filtering=yes
```

- `ether1` = trunk (tagged 20, 30, 99). `ether2`/`ether3` = access ports via `pvid`.
- VLAN 99 lists `bridge1` in `tagged=` so the CPU can reach `MGMT`. Without it, you lose management the moment filtering turns on.
- `frame-types=admit-only-vlan-tagged` on the **bridge** and trunk closes the untagged VLAN 1 path to the CPU.
- `vlan-filtering=yes` is set **last**.

**Hybrid port:** keep `frame-types=admit-all` with `pvid=<untagged-vid>`, and add the port to `tagged=` for each extra tagged VLAN.

**Router-on-a-stick** (no switching): put `/interface/vlan` directly on a physical port that is **not** a bridge member:

```routeros
/interface/vlan/add name=VLAN20 vlan-id=20 interface=ether1
```

## Safe Change Workflow (Avoid Lockout)

Changing `vlan-filtering`, `protocol-mode`, `igmp-snooping` or `dhcp-snooping` **resets the bridge / switch chip** and briefly drops links and MAC-Telnet sessions.

1. Work from an unbridged management port, serial console, or inside **Safe Mode**: `Ctrl+X` (or `F4`) in the CLI (prompt shows `<SAFE>`), or the Safe Mode button in WinBox/WebFig.
2. Create the bridge with `vlan-filtering=no`.
3. Add ports and `pvid`s.
4. Add `/interface/bridge/vlan` entries, including `tagged=<bridge>,...` for the management VLAN.
5. Create `/interface/vlan` and `/ip/address`.
6. Set `vlan-filtering=yes` last.
7. Verify reachability, then press `Ctrl+X` again to commit.

Safe Mode behaviour:
- `Ctrl+D` (or `/quit` then `y`) disconnects and **rolls back** everything done in Safe Mode.
- An unexpected disconnect rolls back automatically after the session times out (about 15 min). To roll back sooner, reconnect, press `Ctrl+X`, answer `u` (unroll).
- Safe Mode history holds 100 changes. Exceeding it **exits Safe Mode automatically and keeps the changes** (no rollback protection after that).

## Gotchas

- **Untagged VLAN 1 leaks to the CPU.** The bridge defaults to `pvid=1 frame-types=admit-all`. A trunk left at `pvid=1` lets untagged frames reach the management plane. Set `frame-types=admit-only-vlan-tagged` on trunks **and** on the bridge.
- **Never list several `vlan-ids` with `untagged=` in one entry:**
  ```routeros
  # WRONG: leaks VLAN 20 untagged out ether3 and VLAN 30 out ether2
  /interface/bridge/vlan/add bridge=bridge1 tagged=ether1 untagged=ether2,ether3 vlan-ids=20,30
  ```
  Use one entry per access VLAN, or omit `untagged=` and rely on `pvid` dynamic membership.
- **`/interface/vlan` goes on the bridge, not on a bridge member.** If `ether1` is in `bridge1`, use `interface=bridge1`. Never add `/interface/vlan` interfaces as bridge ports (breaks RSTP/MSTP and MAC learning).
- **One HW-offloaded bridge per switch chip** (CRS1xx/CRS2xx: up to 7). Extra bridges fall back to the CPU. Check the `H` flag in `/interface/bridge/port/print`.
- **`vlan-filtering=yes` is not offloaded on every chip.** On QCA8337/Atheros (RB3011, hAP ac², hAP ac³, cAP ac, wAP ac, RB450Gx4...) and CRS1xx/CRS2xx it pushes all traffic through the CPU. Those devices use `/interface/ethernet/switch` menus instead. See [switch chip matrix](./references/switch-chips.md).
- **`horizon` disables HW offload** on that port. Prefer switch `port-isolation` (below).
- **L3HW hides traffic from the CPU.** Torch, sniffer and IP firewall do not see HW-routed packets. Disable L3HW (`/interface/ethernet/switch/set 0 l3-hw-offloading=no`) before editing bridge, VLAN, bond or MTU settings. See [L3HW](./references/l3hw.md).
- **Pre-6.41 `master-port` configs** are no longer auto-converted (removed 7.17).

## Port Isolation vs Split Horizon

**Switch port isolation** (private-VLAN style, keeps HW offload):

```routeros
/interface/ethernet/switch/port-isolation/set ether2 forwarding-override=ether1
```

- Works on all switch chips, per port (not per VLAN), only within one switch chip.
- Affects **ingress** only: isolate the other ports too if you need it in both directions.
- On QCA8337/Atheros8327/8316/8227/7240, enabling switch `vlan-mode` overrides port isolation unless ACL rules (`new-dst-ports`) are used.

**Bridge split horizon** (software): ports with the same `horizon` number do not forward to each other. Disables HW offload on those ports.

```routeros
/interface/bridge/port/set [find interface=ether2] horizon=1
```

## Spanning Tree (summary)

- `protocol-mode=none|stp|rstp|mstp`, default `rstp`. `mstp` requires `vlan-filtering=yes`.
- Bridge `priority` in steps of `4096` (`0x0000`..`0xf000`, default `0x8000`), port `priority` in steps of `16` (`0x00`..`0xf0`, default `0x80`). **7.24+** warns on non-compliant values.
- `port-cost-mode=long|short` (**7.13+**, default `long`) auto-derives `path-cost` from link speed.
- Edge protection: `bpdu-guard=yes`, `restricted-role=yes` (root guard), `restricted-tcn=yes`, `auto-isolate=yes`.
- RouterOS does **not** run PVST+.

Full STP/MSTP, IGMP/MLD snooping, DHCP snooping, RA guard, MVRP and MLAG details: [stp-multicast-security.md](./references/stp-multicast-security.md).

## Deprecated / Renamed

| Item | Change | Version |
|---|---|---|
| `master-port` auto-conversion | Removed | 7.17 |
| Bridge Controller Bridge / Port Extender | Removed | 7.18 |
| `/interface/bridge/mdb` `ports=` | Renamed `interface=` | 7.19 |
| `/interface/bridge/mlag` submenu | Moved into `/interface/bridge` properties (downgrade loses MLAG config) | 7.22 |
| `add-dhcp-option82` | Replaced by `dhcp-agent-circuit-id` / `dhcp-agent-remote-id` | 7.23 |
| `last-member-interval` | Renamed `last-member-query-interval` | 7.25 |
| Bridge monitor `path-cost` | Renamed `actual-path-cost` | 7.15 |

## References

- [switch-chips.md](./references/switch-chips.md): which chip offloads what, and the `/interface/ethernet/switch` VLAN config for chips that cannot offload bridge VLAN filtering
- [l3hw.md](./references/l3hw.md): L3 hardware routing offload on Marvell Prestera
- [stp-multicast-security.md](./references/stp-multicast-security.md): STP/RSTP/MSTP, IGMP/MLD snooping, DHCP snooping, RA guard, ARP inspection, IP source guard, MVRP, MLAG
