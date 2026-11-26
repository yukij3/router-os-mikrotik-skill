# IPsec Details

Referenced from `routeros-vpn` SKILL.md.

## Object Model

Configuration flows profile -> peer -> identity -> proposal -> policy:

| Menu | Key properties |
|---|---|
| `/ip/ipsec/profile` | Phase 1 (IKE) crypto |
| `/ip/ipsec/peer` | `exchange-mode=ike2` (recommended), `passive=yes` on responders |
| `/ip/ipsec/identity` | `auth-method` (`pre-shared-key`, `digital-signature`, `eap`, `eap-radius`), `mode-config`, `generate-policy` (`no`, `port-override`, `port-strict`), `policy-template-group` |
| `/ip/ipsec/proposal` | Phase 2 algorithms |
| `/ip/ipsec/policy` | Traffic selectors; static policies support `level=unique` |
| `/ip/ipsec/policy/group` | Groups policy templates; binds to an `xfrm` interface (7.25+) |
| `/ip/ipsec/mode-config` | `responder` (`yes`/`no`), `address-pool`, `split-include`, `system-dns` / `static-dns`, `src-address-list` (dynamic NAT on the initiator) |
| `/ip/ipsec/settings` | Global settings, `ddos-cookie-threshold` (7.22+) |

## Changes by Version

| Version | Change |
|---|---|
| 7.14 | `x25519` DH group |
| 7.16 | Defaults: `dpd-interval` `2m` -> `8s`, `dpd-maximum-failures` `5` -> `4` |
| 7.21 | Post-quantum PPK: `ppk` = `no`, `psk`, `qkd`, `psk-ike-initial` with QKD integration |
| 7.22 | `ddos-cookie-threshold`; `modp8192` removed on MIPS |
| 7.23 | Netlink SA/policy handling |
| 7.25 | Route-based IPsec with `/interface/xfrm` |

## Route-based (XFRM, 7.25+)

```routeros
/interface/xfrm/add name=xfrm1
/ip/ipsec/policy/group/add name=g1 interface=xfrm1
/ip/route/add dst-address=10.20.0.0/24 gateway=xfrm1
```

- `mtu` default `1400`; equal MTU on both sides for OSPF.
- Running (`R`) only once an SA using the interface is up.
- Permit it in the firewall (`LAN` interface list or explicit accept).
- Routes and dynamic routing replace per-subnet policy pairs.

Docs: <https://manual.mikrotik.com/docs/virtual-private-networks/ipsec/route-based-ipsec-xfrm> ; <https://manual.mikrotik.com/docs/cli-reference/interface/xfrm>

## Policy-based Checklist

1. NAT bypass `accept` in `srcnat` **above** masquerade (`place-before=0`).
2. Exempt IPsec traffic from FastTrack: `/ip/firewall/raw` `action=notrack`, or a filter accept before the FastTrack rule.
3. Avoid `modp8192` on MIPS and Ed25519 certificates (unsupported for IPsec).

Docs: <https://manual.mikrotik.com/docs/virtual-private-networks/ipsec>
