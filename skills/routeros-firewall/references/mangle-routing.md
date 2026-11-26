# Mangle Side of Policy Routing

Mangle chains and routing marks for policy-based routing. Referenced from `routeros-firewall` SKILL.md. Verified against RouterOS 7.25.2 stable / 7.24.7 long-term.

This file covers the **mangle** side only. Routing tables, `/routing/rule`, `/routing/settings policy-rules` (7.22+), and route selection are covered by the `routeros-routing` skill. A `new-routing-mark` must name an existing routing table, so create the table first:

```routeros
/routing/table/add name=vpn-mark fib
```

Limits: 4096 unique routing tables and 4096 unique packet marks.

## `passthrough` in Mangle

Every mangle rule has `passthrough=yes|no` (default `yes`). With `yes`, evaluation continues to the next mangle rule after the action; with `no`, mangle evaluation stops for that packet in this chain. Since 7.19, `passthrough` is always shown in export (7.17–7.18 omitted it).

This is a mangle property only. Filter rules use `action=passthrough` instead.

## mark-routing in Mangle Prerouting

Mark packets in `prerouting` to assign them to an alternate routing table:

```routeros
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 \
  action=mark-routing new-routing-mark=vpn-mark passthrough=yes \
  comment="mypolicy-mark-clients"
```

The routing mark is used when RouterOS makes the routing decision for the packet. Whether a mark is consulted before or after other policy rules depends on `/routing/settings policy-rules` — see `routeros-routing`.

## Exempt DNS from Policy Routing

DNS must reach the local router, not the alternate gateway. Add the DNS exemption rule **before** the mark-routing rule:

```routeros
# DNS exempt — MUST be before the mark-routing rule
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 protocol=udp port=53 \
  action=accept comment="mypolicy-exempt-dns"

# Mark remaining traffic
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 \
  action=mark-routing new-routing-mark=vpn-mark passthrough=yes \
  comment="mypolicy-mark-clients"
```

Rule order is critical — DNS exempt must come first (see SKILL.md Rule Ordering section).

## `hotspot=` Matcher

RouterOS hotspot adds a `hotspot=` matcher (IPv4 only) usable in mangle and filter:

```routeros
# Only route AUTHENTICATED hotspot clients through the alternate gateway
# Unauthenticated clients (pre-login) are not affected
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 hotspot=auth \
  action=mark-routing new-routing-mark=vpn-mark passthrough=yes \
  comment="mypolicy-mark-authed"
```

Valid `hotspot=` values: `auth` (authenticated clients), `from-client` (packets from a hotspot client), `to-client` (packets sent to a hotspot client), `local-dst`, `http`.

**Hotspot + mangle routing marks:** by default, mangle routing marks are evaluated before the `local` destination lookup, which can break hotspot portal / HTTP-80 interception. On 7.22+ reorder `/routing/settings policy-rules` so `local` comes before `mangle`, or add `/routing/rule` entries for local hotspot traffic first. See `routeros-routing` and `routeros-hotspot`.

## Generic Policy Routing Pattern (Mangle Side)

Route traffic from one subnet through an alternate gateway:

```routeros
# 1. Routing table (details: routeros-routing)
/routing/table/add name=vpn-mark fib

# 2. DNS exempt (before mark rule)
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 protocol=udp port=53 \
  action=accept comment="mypolicy-exempt-dns-udp"
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 protocol=tcp port=53 \
  action=accept comment="mypolicy-exempt-dns-tcp"

# 3. Mark traffic (authenticated hotspot clients only)
/ip/firewall/mangle/add chain=prerouting \
  src-address=10.20.0.0/24 hotspot=auth \
  action=mark-routing new-routing-mark=vpn-mark passthrough=yes \
  comment="mypolicy-mark"

# 4. NAT for alternate gateway outbound
/ip/firewall/nat/add chain=srcnat action=masquerade \
  out-interface=my-gateway-interface place-before=0 \
  comment="mypolicy-nat"

# 5. Route (start DISABLED — enable after gateway interface is up)
/ip/route/add dst-address=0.0.0.0/0 gateway=my-gateway-interface \
  routing-table=vpn-mark distance=1 disabled=yes

# Enable after gateway is configured:
# /ip/route/enable [find routing-table=vpn-mark]
```

## MSS Clamping

VPN interfaces often have a lower MTU than the WAN interface. Clamp TCP MSS to prevent fragmentation. Always add a `tcp-mss=` range so only **oversized** MSS values are lowered — without it, `change-mss` also rewrites smaller MSS values *upward* to `new-mss`:

```routeros
/ip/firewall/mangle/add chain=forward \
  action=change-mss new-mss=1360 tcp-mss=1361-65535 \
  protocol=tcp tcp-flags=syn out-interface=my-gateway-interface \
  place-before=0 passthrough=yes \
  comment="mypolicy-mss-clamp"
```

Alternative: `new-mss=clamp-to-pmtu` derives the value from the path MTU.

## Other Mangle Changes (7.18+)

- `in-interface`, `in-bridge-port` and `in-bridge` matchers are allowed in `postrouting` (7.18).
- `action=drop` is available in mangle (7.23); `/ipv6/firewall/mangle` also has `drop`, plus `snpt`/`dnpt` (NPTv6).
- IPv6 NAT exists: `/ipv6/firewall/nat` supports `src-nat`, `masquerade`, `dst-nat`, `redirect`, `netmap`.
- `in-bridge-port` no longer requires `use-ip-firewall=yes` on the bridge (7.23 changelog; the 7.25 CLI reference still says it needs `use-ip-firewall` — verify on your version).

## FastTrack Interaction

FastTrack (IPv4, and IPv6 since 7.18) bypasses mangle for fast-pathed flows and only works for the `main` routing table. If a connection is FastTracked, `mark-routing` and other mangle rules are not applied. Put an `action=accept` rule for policy-routed traffic **before** the `fasttrack-connection` rule in filter. See [fasttrack-conntrack.md](./fasttrack-conntrack.md).

## `rp-filter` and Policy Routing

`/ip/settings/set rp-filter=strict` does not work with custom routing tables. Use `rp-filter=loose` or `no` with policy routing, asymmetric paths or VRRP.

## Sources

- [Mangle](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/mangle)
- [Common Firewall Matchers and Actions](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/common-firewall-matchers-and-actions)
- [Policy Routing](https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/policy-routing)
- [/routing/settings](https://manual.mikrotik.com/docs/cli-reference/routing/settings), [/routing/rule](https://manual.mikrotik.com/docs/cli-reference/routing/rule)
- [/ip/settings](https://manual.mikrotik.com/docs/cli-reference/ip/settings)
