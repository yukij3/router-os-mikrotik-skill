# WiFi Sniffer, Torch and Switch-Chip Port Mirroring

Detail for the `routeros-sniffer` skill. Verified against RouterOS 7.25.2 stable /
7.24.7 long-term, 2026-11-26.

## Switch-chip port mirroring (hardware-switched traffic)

`/tool/sniffer`, `/tool/torch` and mangle `sniff-tzsp` only see packets that reach
the CPU. Hardware-offloaded bridged unicast never does. To capture it, mirror the
port in the switch chip to a tap port (or to the CPU, where the sniffer can then see
it).

### Marvell Prestera (`98DXxxxx`, e.g. CRS3xx), 7.15+

Per-port ingress/egress mirroring with multiple source ports:

```routeros
/interface/ethernet/switch/port/set ether2 mirror-egress=yes mirror-ingress=yes
/interface/ethernet/switch/set switch1 mirror-target=ether3
```

| Property | Menu | Notes |
|----------|------|-------|
| `mirror-ingress`, `mirror-egress` | `/interface/ethernet/switch/port` | Per source port (replaces the single `mirror-source` on Prestera since 7.15) |
| `mirror-target` | `/interface/ethernet/switch` | `<port>`, `cpu` or `none` |
| `rspan`, `rspan-ingress-vlan-id`, `rspan-egress-vlan-id` | `/interface/ethernet/switch` | Remote SPAN: carry mirrored traffic in a VLAN |
| `mirror=yes` | `/interface/ethernet/switch/rule` | ACL-based mirroring of matching traffic only |

Version notes: 7.20 added the `mirror-buffers` QoS setting, 7.21 added
`mirror-profile`, and 7.23 made `mirror-buffers=auto` the default.

### Other switch chips (Atheros/QCA, MediaTek, Realtek, 88E)

These use a single `mirror-source` and `mirror-target` under
`/interface/ethernet/switch`.

## WiFi sniffer (`/interface/wifi/sniffer`), 7.16+

This is a monitor-mode 802.11 sniffer for the `wifi` package, separate from
`/tool/sniffer`. Properties:

| Property | Purpose |
|----------|---------|
| `duration` | How long to capture |
| `filter` | libpcap-style filter (`addr1`/`addr2`/`addr3`, `type`/`subtype`, `beacon`, …) |
| `pcap-file`, `pcap-size-limit` | Save to file |
| `stream-address`, `stream-rate` | Stream to a remote receiver |
| `stream-port` | Stream UDP port (7.25+) |
| `show-frame=radiotap` | Radiotap frame display (7.22+) |

It returns an error on radios without `hw-caps=sniffer`. On the receiving side,
the TZSP encapsulation types for 802.11 are `18` (802.11), `119` (Prism),
`126` (Radiotap) and `127` (AVS).

## Torch (`/tool/torch`)

Torch is a real-time per-flow traffic rate monitor. It sees traffic before firewall
filtering and does **not** capture payloads. It can classify flows by IPv4/IPv6 source and
destination, port, protocol, `mac-protocol`, VLAN ID, `mac-address` and DSCP.
Data reporting was improved in 7.19.

Torch has the same blind spots as the sniffer: no hardware-offloaded bridge
traffic and no RoMON frames. It also turns FastTrack/FastPath off while running.
