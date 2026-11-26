---
name: routeros-sniffer
description: "RouterOS packet capture and TZSP streaming for protocol debugging. Use when: capturing packets on RouterOS, setting up /tool/sniffer, streaming live traffic via TZSP, using firewall mangle action=sniff-tzsp, debugging network protocols on MikroTik, receiving TZSP with Wireshark or tshark, saving pcap/pcapng files from RouterOS, sniffing WiFi frames with /interface/wifi/sniffer, watching flows with /tool/torch, mirroring switch-chip ports when offloaded traffic is invisible, or when the user mentions packet sniffer, TZSP, sniff-tzsp, /tool/sniffer, torch, port mirroring, or packet capture on RouterOS."
---

# RouterOS Packet Capture & TZSP Streaming

> Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.
> Features newer than 7.12 are marked with their minimum version.

RouterOS has a built-in packet sniffer (`/tool/sniffer`) and firewall mangle actions that can mirror traffic — either saving to a file on the router or streaming live to a remote host via TZSP (TaZmen Sniffer Protocol). This is the primary way to capture packets on RouterOS since standard tools like `tcpdump` do not exist (see `routeros-fundamentals` skill).

## Why This Matters for Agents

When debugging any network protocol issue on RouterOS, agents should know they can:
1. **Stream live packets** from the router to the host machine via TZSP — no hardware needed if using a CHR VM
2. **Save pcap/pcapng files** on the router's flash and download them for analysis
3. **Use firewall mangle rules** for surgical, per-flow packet mirroring without touching the sniffer config

Combined with a QEMU CHR instance (see `routeros-qemu-chr` skill), this gives agents a complete packet-level debugging workflow with zero physical hardware.

## Method 1: /tool/sniffer (Full Capture Tool)

The built-in sniffer captures packets on specified interfaces with extensive filtering. It supports **three independent output modes** that can be combined:

| Output | Setting | Notes |
|--------|---------|-------|
| Memory buffer | (always on) | Viewable via `quick`, `packet`, `protocol`, `host`, `connection` submenus. Packets available for 10 minutes |
| File on flash | `file-name=capture.pcap` | Always PCAPNG since RouterOS 7.20, even with a `.pcap` name |
| TZSP stream | `streaming-enabled=yes` | UDP to `streaming-server` on `streaming-port` (default 37008) |

### Live TZSP Streaming

```routeros
# Configure sniffer to stream via TZSP to a remote host
/tool/sniffer
set streaming-enabled=yes streaming-server=<RECEIVER-IP> streaming-port=37008

# Optional: filter to a specific interface or protocol
set filter-interface=ether1
set filter-ip-protocol=icmp

# Start capture (runs until stopped or router reboots)
/tool/sniffer/start

# Stop when done
/tool/sniffer/stop
```

`streaming-server` takes a bare IP address (default `0.0.0.0`). It does **not** accept `IP:port`; that gives an `invalid value` error. Set the UDP port with `streaming-port` (default `37008`).

The receiver host runs Wireshark, tshark, or another TZSP-capable tool (see [TZSP receivers reference](./references/tzsp-receivers.md)).

### File-Based Capture

```routeros
# Capture to file on router flash
/tool/sniffer
set file-name=capture.pcap filter-interface=ether1

/tool/sniffer/start
# ... let it capture ...
/tool/sniffer/stop

# Or save the memory buffer to a file manually
# (/flash/ exists only on RAM-root boards; elsewhere use a plain file name)
/tool/sniffer/save file-name=/flash/debug.pcap

# Download via SCP or fetch
# From the host:
# scp admin@<ROUTER-IP>:/flash/debug.pcap .
```

File + streaming can run simultaneously:
```routeros
/tool/sniffer
set file-name=capture.pcap streaming-enabled=yes streaming-server=<RECEIVER-IP>
/tool/sniffer/start
```

### Quick Mode (Interactive CLI)

For quick one-off inspection directly on the router console:

```routeros
# Quick-capture ICMP traffic on ether1
/tool/sniffer/quick ip-protocol=icmp interface=ether1
```

This shows a live scrolling table on the console with source/dest MAC, IP, protocol, and size. Since 7.15, `quick` accepts `rows` to limit the table size.

### Sniffer Filter Properties

Key filter options for `/tool/sniffer/set`:

| Property | Description |
|----------|-------------|
| `filter-interface` | Interface name or `all` (default: `all`) |
| `filter-ip-address` | Up to 16 IP/mask entries |
| `filter-dst-ip-address` / `filter-src-ip-address` | Up to 16 destination / source IP/mask entries |
| `filter-ipv6-address` | Up to 16 IPv6 entries (also `filter-dst-ipv6-address`, `filter-src-ipv6-address`) |
| `filter-mac-address` | Up to 16 `mac/mask` entries (also `filter-dst-mac-address`, `filter-src-mac-address`) |
| `filter-port` | Up to 16 ports (supports `!` negation) |
| `filter-dst-port` / `filter-src-port` | Up to 16 `[!]port` entries |
| `filter-ip-protocol` | Up to 16 protocols (tcp, udp, icmp, etc.) |
| `filter-mac-protocol` | Up to 16 MAC protocols (ip, arp, ipv6, vlan, etc.) |
| `filter-direction` | `any`, `rx`, or `tx` |
| `filter-vlan` | Up to 16 VLAN IDs |
| `filter-cpu` | Filter by CPU number |
| `filter-size` | Packet size `0..65535` or a range |
| `filter-operator-between-entries` | `or` (default) or `and` — logical operator between filter entries |
| `filter-stream` | `yes`/`no` — filter out sniffer's own TZSP packets (default: yes) |
| `memory-limit` | Memory buffer size (default: 100 KiB) |
| `memory-scroll` | `yes` (default) overwrites the oldest buffered packets when `memory-limit` fills |
| `file-limit` | Max file size (default: 1000 KiB) |
| `only-headers` | Save only packet headers, not full payload |
| `max-packet-size` | 7.19+: `2k..64k`, capture more than 2 KiB per packet (jumbo frames). Exact syntax not yet in the official property table |
| `show-frame` | `yes`/`no` (default `no`) |
| `streaming-server` / `streaming-port` | TZSP receiver IP (no port) / UDP port (default 37008) |

**Important:** `filter-stream=yes` (default) excludes the sniffer's own TZSP stream packets from the capture — leave this on to avoid feedback loops.

## Method 2: Firewall Mangle (Targeted Mirroring)

Firewall mangle rules offer **granular per-flow mirroring** using the full firewall matcher. Two sniff-related actions exist:

### action=sniff-tzsp (Stream to Remote TZSP Receiver)

Mirrors matching packets as TZSP to a remote host. Uses the firewall's full matching engine (src/dst address, protocol, port, connection state, interface, etc.):

```routeros
# Mirror all forwarded ICMP to a TZSP receiver
/ip/firewall/mangle
add action=sniff-tzsp chain=forward protocol=icmp \
    sniff-target=<RECEIVER-IP> sniff-target-port=37008 \
    comment="TZSP mirror ICMP to Wireshark"

# Mirror traffic from a specific host
/ip/firewall/mangle
add action=sniff-tzsp chain=forward src-address=192.168.88.100 \
    sniff-target=<RECEIVER-IP> sniff-target-port=37008

# Mirror DNS queries
/ip/firewall/mangle
add action=sniff-tzsp chain=forward protocol=udp dst-port=53 \
    sniff-target=<RECEIVER-IP> sniff-target-port=37008
```

Properties for `sniff-tzsp`:
- `sniff-target` (IP) — destination IP for the TZSP UDP packets
- `sniff-target-port` (port, default 37008) — destination UDP port

(`sniff-id` is **not** a `sniff-tzsp` property; it belongs to CALEA lawful-intercept rules.)

**Key behavior:** `sniff-tzsp` acts like `passthrough` — after matching, the packet continues to the next mangle rule. The original packet is NOT modified or dropped; only a copy is sent as TZSP.

**FastTrack:** FastTracked packets bypass the firewall, including mangle. Established FastTracked flows never hit a `sniff-tzsp` rule. Exclude the flow you are debugging from the FastTrack rule, or use `/tool/sniffer`.

### action=sniff-pc (CALEA)

`action=sniff-pc` sends the matched packet to a remote RouterOS CALEA server. It also behaves like `passthrough`, and it requires the `calea` package. Use `sniff-tzsp` for normal Wireshark debugging.

### Mangle vs /tool/sniffer: When to Use Which

| Scenario | Use |
|----------|-----|
| Capture all traffic on an interface | `/tool/sniffer` |
| Mirror specific flows (by IP, port, protocol) | Mangle `sniff-tzsp` |
| Save pcap file on router flash | `/tool/sniffer` with `file-name` |
| Stream live to Wireshark/tshark | Either — `/tool/sniffer` with `streaming-enabled` or mangle `sniff-tzsp` |
| Multiple independent mirrors to different receivers | Mangle rules (one per target) |
| Quick interactive CLI view | `/tool/sniffer/quick` |

## TZSP Protocol Overview

TZSP is a simple UDP encapsulation — the router wraps the original Ethernet frame in a TZSP header and sends it as a UDP datagram:

```
UDP (port 37008) → TZSP header (4 bytes) → tags (variable) → TAG_END → original Ethernet frame
```

- **Default port:** 37008 (`0x9090`) — not IANA-registered but the RouterOS/Wireshark standard
- **Encapsulation:** `1` Ethernet (the usual case for `/tool/sniffer`), `7` Raw IP, `18` IEEE 802.11, `119` 802.11 Prism, `126` 802.11 Radiotap, `127` 802.11 AVS (values from the Wireshark TZSP dissector)
- **VLAN tags:** before 7.23, streamed TZSP packets could be missing their 802.1Q tag; fixed in 7.23
- **Tags:** Optional metadata (WLAN signal strength, channel, etc.); Ethernet captures usually have no tags
- **Keepalives:** Type 4 (Null) packets sent periodically — no inner frame, filter these when processing

## CHR Testing Pattern

A QEMU CHR instance provides a complete packet capture lab with zero hardware. The free CHR license has a **1 Mbps speed limit** but this is sufficient for protocol debugging and sniffer testing.

```sh
# 1. Boot a CHR instance with port forwarding for REST API and SSH
qemu-system-x86_64 -M q35 -m 256 \
  -drive file=chr.img,format=raw,if=virtio \
  -netdev user,id=net0,hostfwd=tcp::9180-:80,hostfwd=tcp::9122-:22 \
  -device virtio-net-pci,netdev=net0 \
  -display none -serial stdio

# 2. Configure sniffer via REST API once booted
curl -u admin: -X POST http://<router-ip>/rest/tool/sniffer/set \
  -H 'Content-Type: application/json' \
  -d '{"streaming-enabled":"yes","streaming-server":"10.0.2.2","streaming-port":"37008"}'

# 3. Start capture
curl -u admin: -X POST http://<router-ip>/rest/tool/sniffer/start

# 4. Listen for TZSP on the host
tshark -i any -f "udp port 37008" -O tzsp

# 5. Generate test traffic (e.g., ping from the CHR)
curl -u admin: -X POST http://<router-ip>/rest/ping \
  -H 'Content-Type: application/json' \
  -d '{"address":"8.8.8.8","count":"3"}'
```

**QEMU user-mode networking note:** The CHR's default gateway (10.0.2.2) is the host. Use this IP as `streaming-server` when using QEMU `-netdev user`. The host must listen on all interfaces (or 10.0.2.2 specifically) to receive the TZSP packets.

For full QEMU setup details, see the `routeros-qemu-chr` skill. For CHR licensing details (free tier, 60-day trial, speed limits), see [CHR licensing](../routeros-qemu-chr/references/chr-licensing.md).

## Other Capture & Monitoring Tools

| Need | Tool | Notes |
|------|------|-------|
| See hardware-switched traffic | Switch-chip port mirroring (`mirror-target=<port\|cpu>`) | Sniffer/mangle cannot see offloaded bridge traffic |
| 802.11 frames in monitor mode | `/interface/wifi/sniffer` (wifi package, 7.16+) | Radio must have `hw-caps=sniffer` |
| Live per-flow bandwidth, no payload | `/tool/torch` | Sees traffic before firewall filtering |

Syntax and version notes are in [references/other-tools.md](./references/other-tools.md).

## Gotchas

- **Hardware-offloaded bridge traffic** is NOT visible to the sniffer, Torch, or mangle `sniff-tzsp`. Only flooded packets (unknown unicast, broadcast, multicast) reach the CPU. Use switch-chip port mirroring instead
- **Wireless client-to-client** unicast with forwarding enabled is NOT visible
- **RoMON frames are invisible:** with RoMON enabled, received RoMON frames (ethertype `0x88bf`, dst `01:80:c2:00:88:bf`) are not shown by the sniffer or Torch
- **Device-mode (7.17+):** `/tool/sniffer` is gated by `/system/device-mode` property `sniffer` — `no` in `mode=home`, `yes` in `basic`/`advanced`/`rose`. It fails with `failure: not allowed by device-mode` (or `configuration flagged` when `flagged=yes`). Changing device-mode needs a physical power cycle or button press
- **Sniffing disables FastTrack/FastPath:** while `/tool/sniffer` (or Torch, or `mac-scan`) runs, FastTrack/FastPath is off, so CPU load goes up on a busy router
- **Sniffed packets in memory expire after 10 minutes** — save to file or use streaming for persistent capture
- **Memory buffer is small:** with `memory-limit=100KiB` and `memory-scroll=yes` (defaults), older packets are overwritten once 100 KiB fills, often well before 10 minutes
- **PCAPNG only (7.20+):** saved files are always PCAPNG, even if named `.pcap`, and there is no option for legacy PCAP. They carry interface name, direction, nanosecond timestamps, and a per-packet comment with CPU number and fast-path status. Timestamp resolution improved again in 7.25. Convert on the host if an old tool needs PCAP. `traffic-generator` reads PCAPNG since 7.21
- **`streaming-server` is IP only** — `IP:port` is rejected; use `streaming-port`
- **`filter-stream=yes`** (default) is important — without it, the sniffer captures its own TZSP stream packets, creating a feedback loop
- **1 Mbps CHR speed limit** may cause "slow" captures — this is the free license limit, not a sniffer issue. See [CHR licensing](../routeros-qemu-chr/references/chr-licensing.md)
- **`file-limit` should not exceed free memory** when writing to RAM-backed storage — the router may crash or behave unexpectedly

## Cleanup

Always clean up sniffer config and mangle rules after debugging:

```routeros
# Stop sniffer
/tool/sniffer/stop

# Reset sniffer config to defaults
/tool/sniffer
set streaming-enabled=no streaming-server=0.0.0.0 file-name=""

# Remove mangle rules (find by comment)
/ip/firewall/mangle/remove [find comment~"TZSP"]
```

## Additional Resources

**Reference files:**
- For TZSP receiver setup (Wireshark, tshark, tcpdump): see [TZSP receivers reference](./references/tzsp-receivers.md)
- For WiFi sniffer, Torch and switch-chip port mirroring: see [other tools reference](./references/other-tools.md)

**Related skills:**
- For QEMU CHR setup and boot patterns: see `routeros-qemu-chr` skill
- For CHR licensing (free tier, 60-day trial, speed limits): see `routeros-qemu-chr` skill
- For RouterOS CLI/REST basics: see `routeros-fundamentals` skill
- For the `/console/inspect` command tree: see `routeros-command-tree` skill

**MCP tools:**
- For RouterOS docs lookup: use the `rosetta` MCP server tools (`routeros_search`, `routeros_get_page`)
