---
name: routeros-routing
description: "RouterOS v7 routing: static routes, RIB vs FIB, custom routing tables, policy routing with /routing/rule, recursive routes and check-gateway WAN failover, ECMP hashing, VRF and VRF-lite route leaking, v7 routing filter syntax, OSPF v2/v3, BGP (instances, connections, templates, sessions, multipath, unnumbered) and BFD on MikroTik. Use when: adding routes or a second routing table, building WAN failover or load balancing, debugging why a route is inactive or not installed, writing /routing/filter/rule chains, migrating OSPF/BGP/route rules from v6, configuring VRFs or leaking routes between them, or when the user mentions /ip/route, /routing/table, /routing/rule, /routing/route, /routing/settings, policy-rules, target-scope, check-gateway, distance, ECMP, /ip/vrf, /routing/filter, /routing/ospf, /routing/bgp, /routing/bfd, AS-path, route-distinguisher or route check."
---

# RouterOS Routing (v7)

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26. Features newer than 7.12 carry their minimum version inline.

CLI syntax, `[find]`, REST verbs and package install are in `routeros-fundamentals`. Mangle `mark-routing`, DNS exemption before marking, `hotspot=auth`, MSS clamping and FastTrack interaction are in `routeros-firewall` ([mangle-routing.md](../routeros-firewall/references/mangle-routing.md)). This skill owns routing tables, `/routing/rule`, route selection, filters and dynamic protocols.

## Menus: RIB vs FIB

| Menu | Purpose |
|---|---|
| `/ip/route`, `/ipv6/route` | Add/edit static routes; shows basic attributes |
| `/routing/route` | **Read-only** view of all families (`ip4`, `ip6`, `link`, `vpn4`, `vpn6`, `evpn`), filtered routes (`F` flag), `immediate-gw`, `contribution`, protocol attributes |
| `/routing/table` | Custom routing tables (must exist before anything references them) |
| `/routing/rule` | Policy routing rules (v6 `/ip/route/rule` is gone) |
| `/routing/settings` | Routing process settings, `policy-rules` order |
| `/ip/vrf` | VRFs (each creates a matching routing table) |
| `/routing/filter/rule` | Route filter chains for OSPF/BGP/RIP/connected/dynamic |
| `/routing/ospf`, `/routing/bgp`, `/routing/bfd` | Dynamic protocols |

Flags worth knowing in prints: `+` = ECMP route, `F` = filtered, `H` = eligible for switch-chip L3 hardware offload (exclude a static route with `suppress-hw-offload=yes`; IPv6 supported since 7.12). L3HW itself is covered in `routeros-bridge-vlan`.

### Default distances

| Source | Distance |
|---|---|
| Connected | 0 |
| Static | 1 |
| DHCP | 1 (`default-route-distance` on the DHCP client) |
| eBGP | 20 |
| OSPF | 110 |
| RIP | 120 |
| iBGP | 200 |
| BGP VPN leaked routes | 200 |

### Ask the router which route a packet takes (7.18+)

```routeros
/ip/route/check src-ip=192.168.88.10 dst-ip=8.8.8.8
```

Returns `status`, `interface`, `nexthop`.

## Custom Tables and Policy Routing

Create the table with the `fib` flag so its routes install into the kernel FIB (limit 4096 unique tables):

```routeros
/routing/table/add name=myTable fib
/ip/route/add dst-address=0.0.0.0/0 gateway=10.112.0.1 routing-table=myTable
```

Select the table either with a mangle `mark-routing` (see `routeros-firewall`) or with a `/routing/rule`:

```routeros
/routing/rule/add src-address=192.168.10.0/24 action=lookup table=myTable
/routing/rule/add dst-address=8.8.8.8 action=lookup-only-in-table table=myTable
```

`/routing/rule` parameters:

| Parameter | Values / notes |
|---|---|
| `action` | `lookup`, `lookup-only-in-table`, `drop`, `unreachable` |
| `table` | Target routing table |
| `src-address`, `dst-address`, `interface`, `routing-mark` | Matchers |
| `min-prefix` | Ignore routes with prefix length <= N in the looked-up table (Linux `suppress_prefixlength`). `min-prefix=0` suppresses the default route |
| `chain` | Default `user`; set to a built-in chain name to insert right after it |
| `disabled` | `yes`/`no` |

### Decision order: `/routing/settings policy-rules` (7.22+)

Default order: `mangle, vrf-lookup, vrf-unreach, local, user, main`.

Because `mangle` runs **before** `local`, a mangle `mark-routing` that also catches packets for the router's own IPs sends them to the other table and breaks access to the router. Fix one of two ways:

```routeros
# Option 1: exclude local destinations in the mangle rule (firewall side)
#   ... dst-address-type=!local ...
# Option 2: evaluate local/user rules before mangle
/routing/settings/set policy-rules=vrf-lookup,vrf-unreach,local,user,mangle,main
```

### Other `/routing/settings`

| Property | Notes |
|---|---|
| `single-process` | `yes`/`no`; default `yes` on devices with <= 64 MB RAM (7.12+), else `no`. Needs reboot |
| `dynamic-in-chain`, `connected-in-chain` | 7.19+: filter chains applied to dynamic and connected routes |
| `check-gateway-ping-interval` / `-timeout` / `-count` | 7.21+: defaults `10s` / `1s` / `2` |

## Recursive Routes, check-gateway, Failover

A route resolves its `gateway` only through routes whose `scope` <= this route's `target-scope`.

| Route type | `scope` | `target-scope` |
|---|---|---|
| Connected | 10 | 5 |
| OSPF/RIP | 20 | 10 |
| Static | 30 | 10 |
| eBGP | 40 | 10 |
| iBGP | 40 | 30 |

v7 keeps a separate nexthop object per `(address, scope, check-gateway)`, so changing one route's `target-scope` no longer breaks other routes sharing the gateway (a v6 problem).

**Recursive WAN failover** (probe hosts pinned to each ISP, default routes resolve through them):

```routeros
/ip/route/add dst-address=8.8.8.8 scope=10 gateway=10.111.0.1
/ip/route/add dst-address=8.8.4.4 scope=10 gateway=10.112.0.1
/ip/route/add distance=1 gateway=8.8.8.8 target-scope=11 check-gateway=ping
/ip/route/add distance=2 gateway=8.8.4.4 target-scope=11 check-gateway=ping
```

`target-scope=11` (> 10) lets the default route resolve through the probe host routes. If ISP1 drops, pings to 8.8.8.8 fail and the distance-2 route takes over.

`check-gateway` on `/ip/route`: `none | arp | ping | bfd`. It needs a gateway IP; it **cannot** be used with an interface-name gateway on a broadcast network. State changes are logged since 7.22. BFD for static route gateways is listed by MikroTik as not yet supported even though `bfd` appears in the property values; prefer `ping`.

**Interface gateways on Ethernet:** `gateway=ether1` on a broadcast link makes the router ARP for every remote destination and disables `check-gateway`. Use the gateway IP; interface gateways are for point-to-point links and tunnels.

## ECMP

Equal-distance active routes to the same prefix merge into one ECMP route (`+` flag), e.g. two default routes with the same distance. Hashing (7.16+) in `/ip/settings` and `/ipv6/settings`:

| Policy | Hash input |
|---|---|
| `l3` (default) | src/dst IP (IPv6 adds flow label and protocol) |
| `l4` | adds src/dst port and protocol |
| `l3-inner` | inner IP headers of tunnelled traffic |

FastPath caches by destination address, and L3HW switch chips use their own hardware hash (including L4 ports), so results can differ from the configured policy. For BGP ECMP see `multipath` below.

## VRF (summary)

```routeros
/ip/vrf/add name=vrf1 interfaces=ether5 place-before=0
/ip/route/add dst-address=0.0.0.0/0 gateway=172.16.1.1@main routing-table=vrf1
/ping 192.168.88.1 vrf=vrf1
```

- Each VRF creates a routing table of the same name. Entries match **top-down** against the built-in `main` (`interfaces=all`), so put custom VRFs **before** `main`. Max 1024 VRFs (enforced since 7.12).
- **7.14+ firewall change:** a VRF gets a virtual interface (e.g. `vrf1`); firewall `in-interface` matches that VRF interface, not the member ports.
- `<ip>@<vrf>` and `vrf=` select the VRF for gateways and services.

Service VRF support, VRF-lite route leaking with `/routing/bgp/vpn` (7.14+) and details: [vrf.md](./references/vrf.md).

## Routing Filters (v7 syntax)

```routeros
/routing/filter/rule/add chain=myChain \
  rule="if (dst in 192.168.1.0/24 && dst-len>24) {set distance +1; accept} else {reject}"
```

- Script-like `if ( <matchers> ) { <actions> } else { <actions> }`.
- **A chain's default action is `reject`.** A prefix that reaches the end without `accept` is dropped (opposite of firewall chains).
- `chain` and `rule` are required (enforced 7.20).
- AS-path regexps match **per ASN**, not per character. Test with `/routing/filter/test-as-path-regexp`.
- 7.20+: `/routing/filter/filter-wizard` builds v7 rules from v6-style flags; `/routing/filter/sync` recalculates filters.

Matchers, actions, lists and gotchas: [routing-filters.md](./references/routing-filters.md).

## OSPF (summary)

v7 has one `/routing/ospf` menu for v2 and v3. Nothing exists by default:

```routeros
/routing/ospf/instance/add name=v2inst version=2 router-id=1.2.3.4
/routing/ospf/area/add name=backbone area-id=0.0.0.0 instance=v2inst
/routing/ospf/interface-template/add networks=192.168.0.0/24 area=backbone
```

- Default DR priority is `128` (v6 was `1`).
- `originate-default=never|if-installed|always` controls the default route; with `never` a redistributed `0.0.0.0/0` is not advertised, whatever the out-filter says.
- 7.23+: static entries can be added in `/routing/ospf/interface` directly.
- 7.24+: OSPF forces passive mode on VRF interfaces.

## BGP (summary)

7.20+ requires an explicit instance; `instance` is mandatory on connections. Downgrading below 7.20 breaks instance config.

| Menu | Role |
|---|---|
| `/routing/bgp/instance` (7.20+) | `name`, `as`, `router-id`, `vrf`, `routing-table`, `cluster-id`, `multipath` (7.22+), `always-compare-med` (7.25+) |
| `/routing/bgp/template` | Protocol defaults (`default` exists) |
| `/routing/bgp/connection` | Peering: `remote.address`, mandatory `local.role`, `local.address`, `output.network`, filters |
| `/routing/bgp/session` | Read-only live state; `clear`, `refresh`, `resend`, `reset`, `stop` |

- `output.network` takes an **address-list name**, not a CIDR.
- `input.filter` keeps rejected routes in memory (`F` in `/routing/route`); `input.accept-nlri` / `input.filter-nlri` and similar drop them before the RIB.

Connection parameters, features by version, unnumbered BGP and BFD: [ospf-bgp-bfd.md](./references/ospf-bgp-bfd.md).

## BFD (summary)

`/routing/bfd/configuration` is **default-deny**: add an allow entry (by `interfaces`, `addresses`, `address-list`, `vrf`), then set `use-bfd=yes` on the BGP connection or OSPF interface template. Entries are order-sensitive. UDP 3784 (single-hop) and 4784 (multihop). Details in [ospf-bgp-bfd.md](./references/ospf-bgp-bfd.md).

## Gotchas

- **Table must exist first.** Create it in `/routing/table` (with `fib`) before any route, rule or mangle references it.
- **Mangle before local.** See `policy-rules` above when marked traffic can't reach the router itself.
- **Filter chains end in reject.** An `out-filter-chain` or BGP `input.filter` without a final `accept` drops everything.
- **BGP `output.network` is an address list,** and a prefix is advertised only when a matching IGP route exists in the routing table (ORIGIN IGP) (unless `output.network-blackhole=yes`).
- **v6 syntax does not exist:** `/routing/ospf-v3`, `/routing/bgp/peer`, `/ip/route/rule` and old `/routing/filter` flags are gone.
- **VRF firewall matching changed in 7.14:** rules on member interfaces stop matching; use the VRF interface or mangle `mark-connection` on the member.

## Deprecated / Removed

| Item | Change | Version |
|---|---|---|
| `/routing/route` `received-from` | Removed (use `bgp.peer-cache-id` / `belongs-to`) | 7.12 |
| `/ip/settings route-cache` | Removed | 7.13 |
| `fib-reinstall` | Removed | 7.20 |
| `/routing/stats/mem-blocks` | Removed | 7.21 |
| `bgp-as-path-slow-legacy` | Deprecated; use numeric regexp in `bgp-as-path` | v7 |
| BGP without `/routing/bgp/instance` | Instance now explicit and required | 7.20 |
| OSPF `nssa-translate` default | `no` changed to `candidate` | 7.21 |

## References

- [routing-filters.md](./references/routing-filters.md): filter matchers, writable properties, commands, lists, AS-path regexp
- [ospf-bgp-bfd.md](./references/ospf-bgp-bfd.md): OSPF details, BGP connection model and features by version, BFD
- [vrf.md](./references/vrf.md): VRF ordering, firewall behaviour, service VRF support, VRF-lite route leaking

Official docs:
- Routing decision: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/routing-decision>
- Moving from v6 to v7: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/moving-from-rosv6-to-rosv7>
- Policy routing: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/policy-routing>
- `/routing/rule`: <https://manual.mikrotik.com/docs/cli-reference/routing/rule>
- `/routing/settings`: <https://manual.mikrotik.com/docs/cli-reference/routing/settings>
- `/ip/route/check`: <https://manual.mikrotik.com/docs/cli-reference/ip/route/check>
