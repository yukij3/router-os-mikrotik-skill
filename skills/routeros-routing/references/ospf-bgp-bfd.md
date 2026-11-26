# OSPF, BGP and BFD Details

Referenced from `routeros-routing` SKILL.md.

## OSPF (`/routing/ospf`)

One menu for OSPFv2 (`version=2`, IPv4) and OSPFv3 (`version=3`, IPv6). v6 `/routing/ospf-v3` does not exist. No instance or area exists by default.

```routeros
/routing/ospf/instance/add name=v2inst version=2 router-id=1.2.3.4
/routing/ospf/area/add name=backbone area-id=0.0.0.0 instance=v2inst
/routing/ospf/interface-template/add networks=192.168.0.0/24 area=backbone
```

| Topic | Notes |
|---|---|
| DR priority | Default `128` (v6: `1`) |
| `type=ptmp-broadcast` | Compatible with v6 `ptmp` (added to `/routing/ospf/interface` in 7.23.2) |
| `nssa-translate` | Default changed `no` -> `candidate` in 7.21 |
| `/routing/ospf/interface` | 7.23+: static entries can be added directly, bypassing `interface-template`; virtual links use the OSPF interface directly |
| VRF interfaces | 7.24+: forced passive |
| Redistribution | Instance `redistribute=`, `originate-default=` (`never`, `if-installed`, `always`), `out-filter-chain=`, `out-filter-select=` |
| Summarization | `/routing/ospf/area/range` (installs a blackhole route for the range) |
| BFD | `use-bfd=yes` on the interface template |

**Default route gotcha:** with `originate-default=never`, `0.0.0.0/0` is not advertised even if `redistribute` and the out-filter would accept it. Use `if-installed` or `always`.

Over route-based IPsec (`xfrm`, 7.25+), match the MTU on both ends; see `routeros-vpn`.

Docs: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/unicast/ospf>

## BGP (`/routing/bgp`)

### Object model

| Menu | Role |
|---|---|
| `/routing/bgp/instance` (7.20+) | `name`, `as`, `router-id`, `vrf`, `routing-table`, `cluster-id`, `multipath` (7.22+), `always-compare-med` (7.25+) |
| `/routing/bgp/template` | Protocol defaults; a `default` template exists |
| `/routing/bgp/connection` | Peering config; `instance` mandatory (7.20+) |
| `/routing/bgp/session` | Read-only live state; commands `clear`, `refresh`, `resend`, `reset`, `stop`, `dump-saved-advertisements` |
| `/routing/bgp/vpn` | VPNv4/VPNv6 export/import per VRF, also used for VRF-lite leaking (see [vrf.md](./vrf.md)) |

Before 7.20 instances were inferred from `router-id`. Configs using `/routing/bgp/instance` do not survive a downgrade below 7.20.

### Connection parameters

| Parameter | Notes |
|---|---|
| `remote.address` | Peer address. Empty = unnumbered (7.22+) |
| `remote.as` | Optional; learned from the OPEN message unless you want to enforce it |
| `local.role` | Mandatory: `ebgp`, `ibgp`, `ibgp-rr`, ... |
| `local.address` | Replaces v6 `update-source`; an interface name for unnumbered |
| `connect`, `listen` | Active / passive session establishment |
| `multihop` | eBGP multihop |
| `nexthop-choice` | `default`, `force-self`, `propagate` |
| `output.network` | **Address-list name** whose prefixes are advertised; only prefixes present in the routing table are sent unless `output.network-blackhole=yes` |
| `input.filter` | Filter chain; rejected routes stay in memory as `F` in `/routing/route` |
| `input.accept-nlri`, `input.filter-nlri`, `input.accept-communities`, `input.filter-communities` (7.19) | Drop updates **before** they are stored, saving RIB memory |
| `use-bfd` | Enable BFD for this session |

v6 `/routing/bgp/peer` does not exist in v7.

### Features by version

| Version | Feature |
|---|---|
| 7.15 | `vpnv6` initial support |
| 7.20 | Explicit `/routing/bgp/instance`; `evpn` initial support; RFC 5549 IPv4 NLRI with IPv6 next hop |
| 7.21 | Duplicate router-IDs allowed on eBGP (RFC 6286) |
| 7.22 | `multipath=<num>` on the instance installs up to N equal-cost BGP paths; BGP unnumbered |
| 7.25 | `always-compare-med`; unnumbered BGP VRF fix |

### BGP unnumbered (7.22+)

Leave `remote.address` empty, set `local.address=<interface>`, and enable RA on the link so the peer is discovered:

```routeros
/ipv6/nd/prefix/add prefix=none interface=ether1
```

Docs: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/unicast/bgp> ; <https://manual.mikrotik.com/docs/cli-reference/routing/bgp/connection> ; <https://manual.mikrotik.com/docs/cli-reference/routing/bgp/instance>

## BFD (`/routing/bfd`)

- `/routing/bfd/configuration` is **default-deny**: anything not explicitly allowed is forbidden.
- Entries are evaluated in order; put `forbid-bfd=yes` entries before broader allow entries. Match by `interfaces`, `addresses`, `address-list`, `vrf`.
- Enable per protocol: `use-bfd=yes` on `/routing/bgp/connection` or `/routing/ospf/interface-template`.
- Ports: UDP 3784 (single-hop), UDP 4784 (multihop). Allow them in the firewall `input` chain.
- Not supported: echo mode, BFD authentication, BFD for static `/ip/route` gateways (the `check-gateway=bfd` value exists, but MikroTik lists this feature as not yet supported).

Docs: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/unicast/bfd> ; <https://manual.mikrotik.com/docs/cli-reference/routing/bfd/configuration>
