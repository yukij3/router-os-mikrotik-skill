# FastTrack, Hardware Offload and Connection Tracking

Referenced from `routeros-firewall` SKILL.md. Verified against RouterOS 7.25.2 stable / 7.24.7 long-term.

## What FastTrack Bypasses

A FastTracked flow skips:

- firewall filter and mangle rule evaluation
- conntrack processing (the connection stays in the table)
- simple queues and global queue trees
- IPsec policy matching
- VRF assignment
- Hotspot universal client

Consequences:

- Exclude IPsec traffic before the FastTrack rule (`ipsec-policy=in,ipsec` and `ipsec-policy=out,ipsec` with `action=accept`).
- Exclude policy-routed traffic (anything that gets a mangle `mark-routing`) before the FastTrack rule. FastTrack only operates for the `main` routing table.
- Queues that must shape traffic will not see FastTracked packets.

FastTrack does **not** produce `connection-state=untracked`; that state only comes from `/ip/firewall/raw action=notrack`.

### Dummy Rule

Adding a `fasttrack-connection` rule makes RouterOS insert a dynamic `action=passthrough` dummy counter rule at the top of the filter table. It stays until the FastTrack rules are removed or disabled **and** the router reboots.

## IPv4 FastTrack

```routeros
/ip/firewall/filter
add chain=forward action=fasttrack-connection connection-state=established,related
add chain=forward action=accept connection-state=established,related
```

## IPv6 FastTrack and FastPath (7.18+)

RouterOS 7.18 added `/ipv6/firewall/filter action=fasttrack-connection`, IPv6 routing FastPath (`allow-fast-path=yes` by default in `/ipv6/settings`), and IPv6 FastTrack rules in `defconf`.

```routeros
/ipv6 firewall filter
add chain=forward action=fasttrack-connection connection-state=established,related comment="defconf: fasttrack6"
add chain=forward action=accept connection-state=established,related,untracked comment="defconf: accept established,related,untracked"
```

> Caveat: the exact `defconf` IPv6 FastTrack rule on 7.18+ (`established,related` vs `established,related,untracked`, and whether `hw-offload` is set) has not been checked against a live default configuration. Inspect with `/system/default-configuration/print` on the target device if it matters.

Read-only status in `/ipv6/settings`: `ipv6-fasttrack-active`, `ipv6-fasttrack-packets`, `ipv6-fasttrack-bytes`, `ipv6-fast-path-active`.

The Filter and Building Advanced Firewall doc pages still contain older "IPv4 only" / "no fast-track support" remarks for IPv6. The IP Settings page and the 7.18 changelog are authoritative.

## FastTrack Hardware Offload

- `hw-offload` exists on `fasttrack-connection` since 7.6; since **7.21** it defaults to `yes` and is hidden on devices without hardware support.
- Requires L3 hardware offloading on supported Marvell Prestera switch chips (CRS3xx, CRS5xx, CCR2xxx, CRS8xx):
  - `/interface/ethernet/switch set 0 l3-hw-offloading=yes`
  - `/interface/ethernet/switch/l3hw-settings` `fasttrack-hw=yes` (default `yes` when `hw-supports-fasttrack=yes`)
- Ports with `l3-hw-offloading=no` still send the first packets of a flow through the CPU and firewall, then offload the established FastTrack flow.
- Switch ACL rules (`/interface/ethernet/switch/rule`) share TCAM with FastTrack HW offload. Prefer switch rules for stateless drops.

```routeros
/interface/ethernet/switch set 0 l3-hw-offloading=yes
/interface/ethernet/switch/port set [find] l3-hw-offloading=no
/ip/firewall/filter
add action=fasttrack-connection chain=forward connection-state=established,related hw-offload=yes protocol=tcp
add action=fasttrack-connection chain=forward connection-state=established,related hw-offload=no
add action=accept chain=forward connection-state=established,related
```

## Connection Tracking Settings

`/ip/firewall/connection/tracking`:

| Item | Notes |
|---|---|
| `udp-timeout` | Default `30s` since 7.14 (was `10s`) |
| `liberal-tcp-tracking` | `yes`/`no`, default `no` (7.20+). With `no`, an out-of-window TCP segment is `invalid`; with `yes`, out-of-window segments are accepted. Increases evasion exposure — troubleshooting only |
| `total-ip4-entries`, `total-ip6-entries` | Counters added in 7.20 |
| `max-entries` | Scales with RAM, up to `1048576`; conntrack table settings are shared/persistent across IPv4 and IPv6 (reworked in 7.18 and 7.20.2) |

Other conntrack-related behavior:

- `connection-nat-state=ein-snat` / `ein-dnat` (7.12+) match connections handled by NAT `action=endpoint-independent-nat` (UDP only).
- Masquerade purges related conntrack entries when the interface address changes (fixes in 7.20.7 / 7.21).
- `/ip/firewall/raw action=notrack` skips conntrack, including IP defragmentation for those packets.
- `/ip/settings` gained `ipv4-fragment-time` (default `3`s) and `ipv4-high-fragment-thresh` (RAM-scaled) in 7.23.

## Sources

- [Connection tracking](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/connection-tracking)
- [Filter](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/filter)
- [Building Advanced Firewall](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/user-guides/building-advanced-firewall)
- [L3 Hardware Offloading](https://manual.mikrotik.com/docs/bridging-and-switching/l3-hardware-offloading)
- [/ip/settings](https://manual.mikrotik.com/docs/cli-reference/ip/settings), [/ipv6/settings](https://manual.mikrotik.com/docs/cli-reference/ipv6/settings)
