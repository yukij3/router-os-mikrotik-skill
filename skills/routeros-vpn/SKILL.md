---
name: routeros-vpn
description: "RouterOS v7 VPN tunnels on MikroTik: WireGuard (peers, client config and QR, wg-import/wg-export, multi-WAN), IPsec IKEv2 (profiles, peers, identities, proposals, policies, mode-config) and route-based IPsec with /interface/xfrm, OpenVPN server/client (UDP, multiple servers, .ovpn import/export), L2TP/IPsec and L2TPv3, SSTP, ZeroTier, and Back To Home (BTH). Use when: building a site-to-site or road-warrior VPN on RouterOS, choosing between WireGuard, IPsec, OpenVPN, L2TP and SSTP, generating client configs, routing or running OSPF/BGP over IPsec, debugging a tunnel that connects but passes no traffic (NAT bypass, FastTrack), or when the user mentions /interface/wireguard, wg-import, show-client-config, /ip/ipsec, xfrm, ovpn-server, ovpn-client, l2tp-server, sstp-server, /zerotier, /ip/cloud back-to-home or BTH."
---

# RouterOS VPN (v7)

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26. Features newer than 7.12 carry their minimum version inline.

CLI syntax, package install and `device-mode` are in `routeros-fundamentals`. Firewall rules, NAT and mangle are in `routeros-firewall`. Routing tables, VRFs and OSPF/BGP over tunnels are in `routeros-routing`.

## Choosing a Tunnel

| Need | Use | Notes |
|---|---|---|
| Fast site-to-site or road warrior, MikroTik or modern clients | WireGuard | Simplest; client config/QR generated on the router |
| Interop with firewalls/vendors, native OS clients (IKEv2) | IPsec | Policy-based, or route-based with `xfrm` (7.25+) |
| Existing OpenVPN clients | OpenVPN | UDP supported; set ciphers explicitly |
| Windows built-in client through restrictive firewalls | SSTP | PPP over TLS on TCP 443 |
| Legacy L2TP/IPsec clients, L2 pseudowire | L2TP / L2TPv3 | |
| Overlay mesh without public IPs | ZeroTier | Extra package, ARM/ARM64 only |
| Reach a home router with no setup | Back To Home | ARM/ARM64/TILE, uses MikroTik Cloud relay if needed |

## WireGuard

```routeros
/interface/wireguard/add name=wg1 listen-port=13231
/ip/address/add address=10.0.0.1/24 interface=wg1
/interface/wireguard/peers/add interface=wg1 name=laptop \
  public-key="<peer-public-key>" allowed-address=10.0.0.2/32
# Accept the UDP port in the input chain (above your drop rule)
/ip/firewall/filter/add chain=input protocol=udp dst-port=13231 action=accept
```

- Adding the interface auto-generates `private-key` and `public-key`.
- `allowed-address` is mandatory and acts as the routing/ACL table for the peer. Entries must not overlap within one interface (7.24+ warns).
- Peer `name` (7.15+) shows in logs and can be used to reference the peer (7.25+).
- `responder=yes` (7.15+): the peer never initiates handshakes (server side).
- Two WireGuard interfaces cannot share a `listen-port` (7.14+). `listen-port=0` picks a port automatically (7.24+).

Generate a client config or QR from the peer entry (needs `show-sensitive` policy since 7.21):

```routeros
/interface/wireguard/peers/set [find name=laptop] client-address=10.0.0.2/32 \
  client-endpoint=vpn.example.com
/interface/wireguard/peers/show-client-config [find name=laptop]
```

Import/export standard `wg` `.conf` files with `/interface/wireguard/wg-import` and `wg-export` (7.12+). `wg-export` contains keys and peers only: no IP addresses, routes or firewall rules.

Full peer properties, `vrf=`, multi-WAN and HotSpot 2FA: [wireguard.md](./references/wireguard.md).

## IPsec

v7 structure, one object per concern:

| Menu | Role |
|---|---|
| `/ip/ipsec/profile` | Phase 1 (IKE) crypto |
| `/ip/ipsec/peer` | Remote endpoint; `exchange-mode=ike2` recommended, `passive=yes` for responders |
| `/ip/ipsec/identity` | Auth (`pre-shared-key`, `digital-signature`, `eap`, `eap-radius`), `mode-config`, `generate-policy`, `policy-template-group` |
| `/ip/ipsec/proposal` | Phase 2 crypto |
| `/ip/ipsec/policy`, `/ip/ipsec/policy/group` | Which traffic is encrypted |
| `/ip/ipsec/mode-config` | Address/DNS/split-include push for road warriors |

### Route-based IPsec with XFRM (7.25+)

An `xfrm` interface bound to a policy group lets you use plain routes and OSPF/BGP instead of one policy pair per subnet:

```routeros
/interface/xfrm/add name=xfrm1
/ip/ipsec/policy/group/add name=g1 interface=xfrm1
/ip/route/add dst-address=10.20.0.0/24 gateway=xfrm1
```

- Default `mtu=1400`. The interface is `R` (running) only after an SA using it is established.
- Add `xfrm1` to the firewall `LAN` interface list or accept it explicitly.
- For OSPF over `xfrm`, make the MTU equal on both ends.

### Policy-based IPsec gotchas

Tunnel up but no traffic usually means NAT or FastTrack:

```routeros
# NAT bypass ABOVE masquerade
/ip/firewall/nat/add chain=srcnat src-address=192.168.88.0/24 \
  dst-address=10.20.0.0/24 action=accept place-before=0
```

Also exempt IPsec traffic from FastTrack (a `/ip/firewall/raw` `action=notrack` rule, or a filter accept before the FastTrack rule).

Recent changes, mode-config and crypto notes: [ipsec.md](./references/ipsec.md).

## OpenVPN

- UDP and TCP (`protocol=udp`; default is still `tcp`, port 1194), IPv4 and IPv6, `mode=ip` (tun) or `mode=ethernet` (tap).
- 7.17+: several servers under `/interface/ovpn-server/server` (`add name=...`), each with `vrf=`.
- **Defaults are weak:** server cipher `aes128-cbc,blowfish128`, client `blowfish128` + `sha1`. Set `protocol=udp cipher=aes256-gcm auth=sha256` (or `auth=null` with GCM on older clients).
- No LZO compression and no NCP cipher negotiation: the client `.ovpn` must name the cipher.

```routeros
/interface/ovpn-server/server/add name=ovpn-server1 protocol=udp \
  cipher=aes256-gcm auth=sha256 require-client-certificate=yes
```

Route pushing, `.ovpn` export/import, `tls-auth` / `tls-crypt`: [openvpn.md](./references/openvpn.md).

## L2TP

- `/interface/l2tp-server/server`, `/interface/l2tp-client`, `/interface/l2tp-ether`.
- L2TPv2 (PPP) and L2TPv3 (`l2tpv3-ip`, `l2tpv3-udp`) pseudowires via `/interface/l2tp-ether`. Server `accept-proto-version=all|l2tpv2|l2tpv3`.
- `use-ipsec=yes|no|require` with `ipsec-secret=` creates a dynamic transport-mode IPsec peer and policy.
- **Gotcha:** several L2TP/IPsec clients behind the **same NAT** fail with `use-ipsec=yes`. Use static tunnel-mode policies with `level=unique` (or `caller-id-type=number` on the server).
- L2TP IPv6 FastPath: 7.18+.

## SSTP

- `/interface/sstp-server/server`, `/interface/sstp-client`. PPP over TLS, TCP 443.
- `ciphers=aes256-gcm-sha384|aes256-sha` (GCM 7.14+), `tls-version=any|only-1.2`, `pfs=yes|no|required` (`required` = ECDHE only, 7.17+), client `add-sni=yes|no` (7.15+).
- MikroTik-to-MikroTik can run without certificates (anonymous DH, **insecure**). Windows clients need a valid server certificate whose CN/SAN matches the address; on RouterOS clients use `verify-server-certificate=yes` and `verify-server-address-from-certificate=yes`.

## ZeroTier

Needs the extra `zerotier` package (**ARM and ARM64 only**: not x86, CHR or MIPS) **and** `zerotier=yes` in `/system/device-mode` (default `no` in `home`/`basic`, `yes` in `advanced`/`rose`). Package install and device-mode changes: `routeros-fundamentals`.

```routeros
/zerotier/enable zt1
/zerotier/interface/add network=<16-hex-network-id> instance=zt1
/zerotier/peer/print
```

- Interface options: `allow-managed`, `allow-default`, `allow-global`.
- Uses UDP 9993 plus random high ports.
- A self-hosted controller is built in: `/zerotier/controller`, `/zerotier/controller/member`.

## Back To Home (BTH)

WireGuard-based remote access managed by MikroTik Cloud. Built into the base package on **ARM, ARM64 and TILE** (7.12+); not on x86, CHR or MIPS.

```routeros
/ip/cloud/set back-to-home-vpn=enabled
# Wait until /ip/cloud/print shows vpn-status: running
/ip/cloud/back-to-home-user/add name=laptop allow-lan=yes
/ip/cloud/back-to-home-user/show-client-config [find name=laptop]
```

- Creates the `back-to-home-vpn` WireGuard interface (`192.168.216.1/24`, `fc00:0:0:216::1/64`), dynamic input/forward/masquerade rules and the name `<serial>.vpn.mynetname.net`.
- Connects directly when reachable, otherwise through an end-to-end encrypted MikroTik relay (`vpn-prefer-relay-code`, 7.13+).
- Enabling BTH turns on Cloud DDNS when `ddns-enabled=auto` (default since 7.17). See `routeros-maintenance` for `/ip/cloud`.
- Users (`/ip/cloud/back-to-home-user`, renamed from `back-to-home-users` in 7.18): `allow-lan=no` by default (internet only, via the `back-to-home-lan-restricted-peers` address list); `expires` cannot be edited after creation; `client-dns` and `client-allowed-address` (7.25+); `file-access=disabled|read-only|full` with `file-access-path`.
- `show-client-config` needs `[find ...]` or an item number. A bare name is a syntax error.
- **`back-to-home-vpn=revoked-and-disabled` is destructive:** it deletes all BTH users, keys and the UDP port. It is not a pause switch.
- **File share (7.18+):** `/ip/cloud/back-to-home-file` shares files or directories over HTTPS on `*.routingthecloud.net` (Let's Encrypt DNS-01 cert, relay when 443 is busy or blocked), with optional `allow-uploads=yes` (resumable 7.25+) and `expires`.

## Gotchas

- **WireGuard `vrf=` (7.21+, CLI only)** moves the encrypted UDP socket into a VRF, not the plaintext `wg` interface. Put the interface in a VRF with `/ip/vrf`.
- **IPsec NAT bypass and FastTrack:** see above. Without both, the tunnel establishes and traffic is masqueraded or fast-tracked past IPsec.
- **IPsec crypto:** `modp8192` was removed on MIPS (7.22); Ed25519 certificates are not supported for IPsec.
- **OpenVPN defaults:** TCP plus `blowfish128`/`aes128-cbc` unless you change them.
- **L2TP/IPsec behind one NAT:** dynamic policies collide; use `level=unique` static policies.
- **BTH `revoked-and-disabled`:** wipes all users.

## Deprecated / Renamed

| Item | Change | Version |
|---|---|---|
| `/ip/cloud/back-to-home-users` | Renamed `/ip/cloud/back-to-home-user` | 7.18 |
| IPsec `dpd-interval` / `dpd-maximum-failures` defaults | `2m`/`5` changed to `8s`/`4` | 7.16 |
| OpenVPN LZO compression | Not supported | v7 |

## References

- [wireguard.md](./references/wireguard.md): peer properties, client config generation, import/export, VRF, multi-WAN, HotSpot 2FA
- [ipsec.md](./references/ipsec.md): object model details, mode-config, version changes, XFRM notes
- [openvpn.md](./references/openvpn.md): ciphers, route pushing, `.ovpn` export/import, TLS keys

Official docs:
- WireGuard: <https://manual.mikrotik.com/docs/virtual-private-networks/wireguard>
- IPsec: <https://manual.mikrotik.com/docs/virtual-private-networks/ipsec>
- Route-based IPsec (XFRM): <https://manual.mikrotik.com/docs/virtual-private-networks/ipsec/route-based-ipsec-xfrm>
- OpenVPN: <https://manual.mikrotik.com/docs/virtual-private-networks/openvpn>
- L2TP: <https://manual.mikrotik.com/docs/virtual-private-networks/l2tp>
- SSTP: <https://manual.mikrotik.com/docs/virtual-private-networks/sstp>
- ZeroTier: <https://manual.mikrotik.com/docs/virtual-private-networks/zerotier>
- Back To Home: <https://manual.mikrotik.com/docs/network-management/cloud/back-to-home>
