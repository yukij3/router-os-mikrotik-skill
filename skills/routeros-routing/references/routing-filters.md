# Routing Filters (`/routing/filter`)

v7 replaced the v6 flag-based filter rules with a small rule language. Referenced from `routeros-routing` SKILL.md.

## Rule Syntax

```routeros
/routing/filter/rule/add chain=myChain \
  rule="if (dst in 192.168.1.0/24 && dst-len>24) {set distance +1; accept} else {reject}"
```

- Form: `if ( <matchers> ) { <actions> } else { <actions> }`.
- **Default action at the end of a chain is `reject`.**
- `chain` and `rule` are required on `/routing/filter/rule`; `chain` and `list` are required on lists (enforced 7.20).
- Chains are referenced from protocols: OSPF `out-filter-chain`, BGP `input.filter`, `/routing/settings dynamic-in-chain` / `connected-in-chain` (7.19+).

## Read-only Matchers

| Matcher | Values / notes |
|---|---|
| `dst`, `dst-len` | Prefix and prefix length |
| `protocol` | `bgp`, `connected`, `dhcp`, `fantasy`, `modem`, `ospf`, `rip`, `static`, `vpn` |
| `afi` | `ipv4`, `ipv6`, `l2vpn`, `l2vpn-cisco`, `vpnv4`, `vpnv6` |
| `active` | Route is active |
| `bgp-as-path`, `bgp-path-len` | AS path regexp, path length |
| `bgp-communities`, `bgp-ext-communities`, `bgp-large-communities` | Community matching |
| `rpki` | `invalid`, `unknown`, `valid`, `unverified` |
| `ospf-type` | `ext1`, `ext2`, `inter`, `intra`, `nssa1`, `nssa2` |
| `ospf-metric`, `origin`, `rtab`, `vrf` | |

## Writable Properties (`set` / `unset`)

| Group | Properties |
|---|---|
| General | `distance`, `scope`, `scope-target`, `pref-src`, `blackhole`, `suppress-hw-offload` |
| Gateway | `gw`, `gw-ll`, `gw-interface`, `gw-check` (`none`, `arp`, `icmp`, `bfd`, `bfd-mh`) |
| BGP | `bgp-weight`, `bgp-local-pref`, `bgp-med`, `bgp-out-med`, `bgp-igp-metric`, `bgp-path-prepend`, `bgp-path-peer-prepend`, `bgp-origin`, `bgp-communities`, `bgp-ext-communities`, `bgp-large-communities` |
| OSPF external | `ospf-ext-type`, `ospf-ext-metric`, `ospf-ext-tag`, `ospf-ext-fwd` |

## Commands

`accept`, `reject`, `return`, `jump <chain>`, `set <prop> <val>`, `unset <prop>`, `append`, `delete`, `filter`, `rpki-verify <group>`.

## Lists and Select Rules

- `/routing/filter/num-list`
- `/routing/filter/community-list`, `community-ext-list`, `community-large-list`
- `/routing/filter/select-rule`: controls which candidate routes are selected for export (e.g. `do-where=get_active`)

## Address-List Matching Gotcha

`dst in myList` matches exact prefixes in the address list **and** `/32` host routes inside listed ranges. To exclude host routes:

```routeros
/routing/filter/rule/add chain=myChain rule="if (dst in myList && dst-len < 32) {accept}"
```

## AS-Path Regexp

- Matches per ASN (Juniper style), **not** per character as in v6/Cisco.
- Test before deploying:

```routeros
/routing/filter/test-as-path-regexp regexp="..." as-path="1234,5678"
```

- `^$` cannot match an empty AS path; use `bgp-path-len == 0`.
- `bgp-as-path-slow-legacy` is deprecated; use the numeric regexp in `bgp-as-path`.

## Migration Helpers (7.20+)

- `/routing/filter/filter-wizard` generates v7 rules from v6-style flags, e.g. `action=accept chain=vpn-in afi=vpnv4 set-bgp-ext-communities=rt:2:2`.
- `/routing/filter/sync` recalculates filters.

Official docs: <https://manual.mikrotik.com/docs/cli-reference/routing/filter/rule>
