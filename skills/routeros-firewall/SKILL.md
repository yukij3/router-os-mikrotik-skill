---
name: routeros-firewall
description: "RouterOS v7 firewall: filter, NAT, mangle, raw, connection tracking, FastTrack (IPv4 and IPv6, hardware offload), address-lists, interface-lists, DDoS/DoS protection, Layer7/tls-host matching and socksify. Use when: writing firewall rules in RouterOS, configuring NAT or port forwarding, setting up address-lists or interface-lists, writing idempotent firewall scripts, configuring DNS redirect, FastTrack, raw/notrack or DDoS rules, marking traffic in mangle, or when the user mentions /ip/firewall, /ipv6/firewall, chain=forward, chain=input, connection-state, fasttrack-connection, address-list, interface-list, notrack, socksify, or layer7-protocol on MikroTik."
---

# RouterOS Firewall

Verified against RouterOS **7.25.2** stable / **7.24.7** long-term (checked 2026-11-26). Features newer than 7.12 are tagged with their minimum version.

## Rule Ordering — Sequential, Not Priority-Based

Rules are evaluated **top-to-bottom** within a chain — first terminating match wins. This is the biggest source of iptables confusion.

- `place-before=0` inserts at the top; default `add` appends at the bottom
- An `action=accept` rule must appear BEFORE any `action=drop` for the same traffic
- **Non-terminal actions do NOT stop evaluation:** `action=add-src-to-address-list`, `action=add-dst-to-address-list`, `action=log` and `action=passthrough` continue to the next rule. A `drop` rule below an `add-src-to-address-list` will still fire.
- In **filter**, `passthrough` is an *action* (`action=passthrough`, counts packets and moves on). In **mangle**, `passthrough=yes|no` is a *rule property* (default `yes`) that decides whether evaluation continues after a marking action. There is no `passthrough=` property on filter rules.

```routeros
# WRONG — drop fires before accept can match
/ip/firewall/filter/add chain=input action=drop
/ip/firewall/filter/add chain=input src-address=10.0.0.1 action=accept

# CORRECT — accept first, drop catches the rest
/ip/firewall/filter/add chain=input src-address=10.0.0.1 action=accept place-before=0
/ip/firewall/filter/add chain=input action=drop
```

## Address-Lists as Dynamic Selectors

LLMs rarely suggest this pattern — they write one rule per IP address instead. Address-lists scale to hundreds of IPs with a single firewall rule.

```routeros
# Build the list (static or dynamic with auto-expiry)
/ip/firewall/address-list/add list=trusted-mgmt address=192.168.1.0/24
/ip/firewall/address-list/add list=trusted-mgmt address=10.0.0.5 timeout=1h

# One rule handles all list members
/ip/firewall/filter/add chain=input src-address-list=trusted-mgmt action=accept \
  comment="myapp-accept-mgmt"
```

Dynamic entries with `timeout=` expire automatically — the primary pattern for DoS blacklists.

**Persistence (RAM vs disk):**
- Static entries without `timeout`, and rule-added entries with `address-list-timeout=none-static`, are saved to disk and appear in `export`.
- Entries with `timeout=`, or rule-added with `address-list-timeout=none-dynamic`, live in RAM and are lost on reboot.
- `none-dynamic` / `none-static` are available since **7.17** (IPv6 address-lists also gained `creation-time` in **7.14**).

**DNS-name entries:** an address-list entry can be a domain name; RouterOS resolves it and keeps dynamic IP entries. Since **7.21**, when several names resolve to the same IP, the highest DNS TTL is used as the dynamic entry timeout.

**Automatic population (7.17+):** `/ip/dhcp-server` and `/ipv6/dhcp-server` have `address-list=` (adds bound lease addresses); `/ipv6/dhcp-client` has `prefix-address-list=`. PPP supports `MT-Address-List` for IPv6 DHCPv6+RADIUS since **7.24**.

## Interface-Lists as Rule Selectors

`in-interface-list=` / `out-interface-list=` — powerful RouterOS pattern LLMs never propose. Eliminates duplicate rules when multiple interfaces serve the same role.

```routeros
# Define the group once
/interface/list/add name=WAN
/interface/list/member/add list=WAN interface=ether1
/interface/list/member/add list=WAN interface=pppoe-out1

# One rule applies to all WAN interfaces
/ip/firewall/filter/add chain=input in-interface-list=WAN action=drop \
  comment="myapp-drop-all-wan"
```

- Predefined lists: `all`, `none`, `dynamic`, `static`.
- A list can `include`/`exclude` other lists; processing order is `include` → `exclude` → static members.
- Maximum 244 interface-lists (enforced since **7.21**).
- Adding a **bridge** to a list is NOT the same as adding its ports. `in-interface-list=` matches the bridge interface; to match the physical port use `in-bridge-port=` / `in-bridge-port-list=`. Since **7.23**, `in-bridge-port` no longer requires `use-ip-firewall=yes` on the bridge (7.23 changelog; the 7.25 CLI reference still says it needs `use-ip-firewall` — verify on your version).

## Comment-as-Tag Pattern (Idempotent Scripts)

RouterOS has no "upsert" — re-running a script without cleanup creates duplicate rules. Use a comment prefix as a tag:

```routeros
# Remove only rules we own — preserves rules from other tools
/ip/firewall/filter/remove [find comment~"myapp-"]
/ip/firewall/address-list/remove [find comment~"myapp-"]
/ip/firewall/nat/remove [find comment~"myapp-"]

# Add with consistent tag — readable in /print output
/ip/firewall/filter/add chain=input src-address-list=trusted-mgmt action=accept \
  comment="myapp-accept-mgmt"
/ip/firewall/filter/add chain=input in-interface-list=WAN action=drop \
  comment="myapp-drop-wan"
```

**Never use `remove [find dynamic=no]`** — this deletes ALL static rules, including those added by other management tools or by `defconf`. Bulk removes silently break remote management.

## Connection State and Tracking

RouterOS `connection-state=` is not iptables `-m state`:

```routeros
/ip/firewall/filter/add chain=input connection-state=established,related action=accept
/ip/firewall/filter/add chain=input connection-state=invalid action=drop
```

RouterOS states: `new`, `established`, `related`, `invalid`, `untracked`.

`untracked` matches packets explicitly marked via `/ip/firewall/raw action=notrack` — it does NOT match FastTrack flows. Note that `notrack` also skips conntrack defragmentation/reassembly for those packets.

Conntrack changes worth knowing:
- Default UDP timeout is **30s** since **7.14** (was 10s).
- `connection-nat-state` accepts `ein-snat` / `ein-dnat` (7.12+), pairing with NAT `action=endpoint-independent-nat` (UDP only).
- `/ip/firewall/connection/tracking` `liberal-tcp-tracking` (default `no`, **7.20+**) — troubleshooting only; it weakens invalid-packet detection.

Details (counters, `max-entries`, FastTrack bypass list, hardware offload) are in [references/fasttrack-conntrack.md](./references/fasttrack-conntrack.md).

## FastTrack (IPv4 and IPv6)

FastTrack sends established flows through a fast path that **bypasses** firewall filter, conntrack processing, simple queues and global queue trees, IPsec, VRF assignment, **mangle**, and the Hotspot universal client.

```routeros
/ip/firewall/filter
add chain=forward action=fasttrack-connection connection-state=established,related
add chain=forward action=accept connection-state=established,related
```

- **IPv6 FastTrack exists since 7.18** (`/ipv6/firewall/filter action=fasttrack-connection`), plus IPv6 routing FastPath, and `defconf` includes an IPv6 FastTrack rule. Some MikroTik doc pages still say IPv6 has no FastTrack — that text is stale.
- FastTrack only works for traffic routed in the `main` table. Exclude IPsec (`ipsec-policy=in,ipsec` / `out,ipsec`) and policy-routed (mangle-marked) traffic with accept rules **before** the FastTrack rule.
- Since **7.21**, `hw-offload=yes` is the default on `fasttrack-connection` (hidden on hardware without support).
- Enabling FastTrack adds a dynamic `action=passthrough` dummy counter rule at the top of filter; it only disappears after the FastTrack rules are removed/disabled **and** the router is rebooted.

## NAT Patterns

```routeros
# Port forward (dst-nat)
/ip/firewall/nat/add chain=dstnat \
  dst-port=8080 protocol=tcp in-interface=ether1 \
  action=dst-nat to-addresses=192.168.1.10 to-ports=80 \
  comment="portfwd-web"

# Force DNS through router (prevents DNS bypass)
/ip/firewall/nat/add chain=dstnat action=redirect \
  in-interface-list=LAN dst-port=53 protocol=udp to-ports=53 \
  comment="force-dns-udp"
/ip/firewall/nat/add chain=dstnat action=redirect \
  in-interface-list=LAN dst-port=53 protocol=tcp to-ports=53 \
  comment="force-dns-tcp"

# Masquerade outgoing traffic
/ip/firewall/nat/add chain=srcnat action=masquerade \
  out-interface=ether1 comment="nat-wan"
```

- NAT chains in v7: `srcnat`, `dstnat`, plus `input` and `output`.
- **NAT only applies to new connections.** Editing NAT rules does not change already-tracked connections — remove them (`/ip/firewall/connection/remove`) or wait for timeout.
- **Masquerade on multi-WAN:** masquerade purges its tracked connections on link down / address change (purge-on-address-change fixes in 7.20.7 / 7.21). After a failover and recovery, connections can leak private source IPs out the primary link — use a blackhole route or `src-nat` with a fixed address where this matters.
- Random external port allocation is supported since **7.17**.
- `tls-host` was removed from NAT tables in **7.16**; use it in filter/mangle.

### Socksify (7.20+)

Transparently redirect TCP connections to a SOCKS5 server:

```routeros
/ip socksify
add connection-timeout=10 disabled=no name=TOR_socksify socks5-port=9050 socks5-server=10.0.0.2
/ip firewall filter
add action=accept chain=input dst-port=952 protocol=tcp src-address=192.168.88.0/24
/ip firewall nat
add action=socksify chain=dstnat dst-port=80,443 protocol=tcp socksify-service=TOR_socksify src-address=192.168.88.0/24
```

`/ip/socksify` properties: `name`, `socks5-server` (IPv4 only), `socks5-port` (default `1080`), `port` (local listener, default `952` — must be allowed in `input`), `socks5-user`, `socks5-password`, `connection-timeout` (default `60`), `disabled` (default `yes`).

## Raw and DDoS

`/ip/firewall/raw` runs in `prerouting`/`output` **before connection tracking** — the cheapest place to drop known-bad traffic during a flood. The official DDoS pattern detects with `dst-limit` in filter and drops in raw. See [references/dos-protection.md](./references/dos-protection.md).

## Layer7 Protocol (L7) and `tls-host`

L7 matches unencrypted payload content with a POSIX regex — CPU-intensive, use sparingly:

```routeros
/ip/firewall/layer7-protocol/add \
  name=captive-detect \
  regexp="^.*(gstatic|connectivitycheck|generate_204).*$" \
  comment="android-captive-portal"
```

- L7 inspects only the first 10 packets or 2 KB of a connection, and must see **both directions** (use `forward`, or paired `input`/`output` rules).
- HTTPS, QUIC and HTTP/3 make L7 largely useless for modern traffic.
- Alternation: when the pattern is anchored, keep alternatives inside one group: `^(a|b|c)$`. In `^a|b|c$`, `^` binds only to `a` and `$` only to `c`, because `|` has the lowest precedence. Unanchored, `(a)|(b)|(c)` and `(a|b|c)` are equivalent.
- `tls-host=` is a glob matcher on the TLS SNI. It fails when the ClientHello is split across TCP segments or encrypted (ECH).

## Mangle (Marks and Policy Routing)

Mangle marks connections/packets/routing, changes MSS/TTL/DSCP, and (since **7.23**) supports `action=drop`. The mangle side of policy routing (`mark-routing`, `hotspot=` matcher, MSS clamping, FastTrack exclusions) is in [references/mangle-routing.md](./references/mangle-routing.md). Routing tables, `/routing/rule` and `/routing/settings policy-rules` belong to the `routeros-routing` skill.

## Deprecated / Removed Matchers

| Item | Status |
|---|---|
| `src-address-type` / `dst-address-type` values `prohibited`, `unreachable` (IPv4) | Removed in 7.13 (some doc tables still list them) |
| `tls-host` in NAT | Removed from NAT in 7.16 |
| `p2p=` matcher | Deprecated in docs — modern P2P is encrypted, matcher is ineffective (IPv4 only) |

## IPv6 Firewall Notes

- `/ipv6/firewall` is a **separate** rule set — `/ip/firewall` rules never apply to IPv6.
- `reject-with` gained `icmp-err-src-routing-header` and `icmp-headers-too-long` (7.13).
- IPv6 policy routing fixed in 7.19.4; IPv6 interface matchers with VRF interfaces fixed in 7.20.

## Common LLM Mistakes

| Mistake | Correct RouterOS behavior |
|---------|--------------------------|
| Using `priority=` or rule weight | Rules are sequential — order is position, not weight |
| Writing one rule per IP address | Use `src-address-list=` or `dst-address-list=` |
| `remove [find dynamic=no]` in scripts | Tag-based: `remove [find comment~"prefix-"]` only |
| Forgetting `place-before=` on accept rules | Default appends — accept rules below drops never fire |
| `connection-state=new,established` as an iptables-style state list | Valid states: `new`, `established`, `related`, `invalid`, `untracked` |
| `passthrough=yes` on a filter rule | Filter uses `action=passthrough`; `passthrough=` is a mangle property |
| Assuming `action=log` stops evaluation | Non-terminal actions continue to next rule — a `drop` below still fires |
| Combining FastTrack + mangle routing marks on the same flows | FastTrack bypasses mangle — exclude marked traffic before the FastTrack rule |
| "IPv6 has no FastTrack" | IPv6 FastTrack exists since 7.18 |
| Expecting a new NAT rule to affect live connections | Only new connections; clear conntrack entries if needed |
| Anchored L7 regex `^a|b|c$` | Use `^(a|b|c)$` — anchors bind to one branch otherwise |
| One firewall rule per interface | Use `in-interface-list=` with a named interface list |
| Adding a bridge to an interface-list to match its ports | Use `in-bridge-port=` / `in-bridge-port-list=` |
| IPv6 traffic handled by `/ip/firewall` | IPv6 uses a **separate** `/ipv6/firewall` — rules do not apply cross-protocol |
| DDoS drops in filter | Detect in filter, drop in `/ip/firewall/raw` before conntrack |

## Additional Resources

**Related skills:**
- `routeros-fundamentals` — RouterOS CLI syntax, REST API, scripting basics
- `routeros-routing` — routing tables, `/routing/rule`, `/routing/settings policy-rules`
- `routeros-hotspot` — hotspot chain interaction with firewall, walled garden

**Reference files in this skill:**
- [references/mangle-routing.md](./references/mangle-routing.md) — mangle side of policy routing, `hotspot=` matcher, MSS clamping
- [references/fasttrack-conntrack.md](./references/fasttrack-conntrack.md) — FastTrack (IPv4/IPv6), hardware offload, conntrack settings
- [references/dos-protection.md](./references/dos-protection.md) — official DDoS raw pattern, `psd`, `tarpit`, `connection-limit`

**MCP tools:**
- `rosetta` MCP — `/ip/firewall` command tree inspection (`routeros_search`, `routeros_get_page`)

**MikroTik docs:**
- [Filter](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/filter)
- [NAT](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/nat)
- [Mangle](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/mangle)
- [Raw](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/raw)
- [Address-lists](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/address-lists)
- [Layer7](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/layer7)
- [Common Firewall Matchers and Actions](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/common-firewall-matchers-and-actions)
- [Connection tracking](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/connection-tracking)
- [Building Advanced Firewall](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/user-guides/building-advanced-firewall)
- [Interface lists (CLI)](https://manual.mikrotik.com/docs/cli-reference/interface/list/)
- [Socksify](https://manual.mikrotik.com/docs/network-management/socks/socksify)
