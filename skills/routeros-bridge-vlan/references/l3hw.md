# L3 Hardware Offloading (L3HW)

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.

## Supported devices

Only Marvell Prestera: **CRS3xx, CRS4xx, CRS5xx, CRS8xx, CCR2116, CCR2216, RDS2216**.
Not supported: RB5009, RB4011, L009, CCR2004, CRS1xx/CRS2xx.

## Enable

```routeros
/interface/ethernet/switch/set 0 l3-hw-offloading=yes
/interface/ethernet/switch/port/set [find] l3-hw-offloading=yes
```

Set `l3-hw-offloading=no` on WAN/uplink ports that need CPU firewall/NAT. FastTrack connections can still be HW-offloaded on `98DX8xxx` / `98DX4xxx` / `98DX325x` with `action=fasttrack-connection ... hw-offload=yes`; not on `98DX3xxx` / `98DX2xxx`, and currently not on `98DX7335` (CRS8xx).

## Settings and monitoring

`/interface/ethernet/switch/l3hw-settings`: `ipv6-hw=yes` (IPv6 routing offload), `fasttrack-hw`, `icmp-reply-on-error`, `neigh-dump-retries`.

Monitor with `/interface/ethernet/switch/l3hw-settings/monitor` and its `advanced/monitor` counterpart.

## Per-VLAN control (7.21+)

`/interface/vlan` has `l3-hw-offloading=yes|no` (default `yes`) and an `H` flag. Requires the VLAN to sit on a HW-offloaded `vlan-filtering=yes` bridge. Exception: `98DX7xxx` (CRS8xx) also offloads VLANs placed directly on Ethernet ports (7.24+).

## HW VRF and VXLAN

- **HW VRF** (7.23+/7.24+): `98DX8xxx`, `98DX4xxx`, `98DX325x`, `98CX8410`, `98DX7xxx`, plus ACL `new-vrf`. Not on `98DX3xxx` / `98DX2xxx`.
- **VXLAN offload** (7.18+): `98DX8xxx` / `98DX4xxx` / `98DX325x`. Not on `98DX3xxx` / `98DX2xxx`, and not with MLAG.

## Route suppression

When the route table exceeds HW capacity, keep selected routes in software:

```routeros
/ip/route/set [find dst-address=10.0.0.0/8] suppress-hw-offload=yes
```

or in a routing filter: `set suppress-hw-offload yes; accept`. The `H` flag on a route means **eligible** for offload, not necessarily offloaded.

## Gotchas

- HW-routed packets bypass the CPU: Torch, sniffer and IP firewall do not see them. Use ACL `redirect-to-cpu=yes` or `copy-to-cpu=yes` when you need visibility.
- MAC-Telnet (UDP 20561) is dropped on `98DX8xxx` / `98DX4xxx` / `98DX325x` under L3HW unless an ACL allows it.
- Disable L3HW globally (`/interface/ethernet/switch/set 0 l3-hw-offloading=no`) before bridge, VLAN, bond or MTU edits, then re-enable.
