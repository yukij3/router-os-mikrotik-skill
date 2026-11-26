# VRF (`/ip/vrf`)

Referenced from `routeros-routing` SKILL.md.

## Creation and Ordering

```routeros
/ip/vrf/add name=vrf1 interfaces=ether5 place-before=0
```

- Adding a VRF creates a routing table with the same name.
- VRF entries match **top-down**. The built-in `main` VRF has `interfaces=all`, so a custom VRF placed after it never matches. Use `place-before=0` or `/ip/vrf/move`.
- Max 1024 VRFs (enforced since 7.12).
- Default distance of BGP VPN leaked routes: `200`.

## Firewall Behaviour (7.14+)

Interfaces assigned to a VRF sit behind a virtual VRF interface (e.g. `vrf5`). Firewall rules match that VRF interface (`in-interface=vrf5`), **not** the member interfaces. Before 7.14 member interfaces matched.

For per-member filtering, mark connections in mangle on the member interface (`in-interface=ether15 action=mark-connection ...`) and filter on the connection mark. Mangle details live in `routeros-firewall`.

## Selecting a VRF: `vrf=` and `@vrf`

Services and lookups default to `main`. Use `vrf=<name>` or `<ip>@<vrf>`:

```routeros
/ip/route/add dst-address=0.0.0.0/0 gateway=172.16.1.1@main routing-table=vrf1
/ping 192.168.88.1 vrf=vrf1
/ip/dns/set servers=8.8.8.8@vrf1
```

VRF support added in recent versions:

| Feature | VRF support | Version |
|---|---|---|
| DHCPv4 relay | `dhcp-server-vrf` | 7.15 |
| `/ip/service` | `vrf=` on all services; `ftp` gained it in 7.17 | 7.17 for ftp |
| OpenVPN server | `vrf=` | 7.17 |
| `/system/logging/action` | `vrf=` | 7.19.6 |
| `/ip/dns` | `vrf=` and `servers=<ip>@<vrf>` | 7.21 |
| WireGuard | `vrf=` for the encrypted UDP socket only | 7.21 |
| TFTP | VRF support | 7.25 |

OSPF forces passive mode on VRF interfaces (7.24+).

## VRF-lite Route Leaking without a BGP Peer (7.14+)

`/routing/bgp/vpn` imports/exports routes between VRFs using route targets; no BGP session is needed:

```routeros
/routing/bgp/vpn/add name=vrf1-vpn vrf=vrfTest1 route-distinguisher=1.2.3.4:1 \
  export.redistribute=connected .route-targets=1:1 import.route-targets=1:2 \
  label-allocation-policy=per-vrf
```

Usually a second entry for the other VRF uses the mirrored route targets. Leaked routes get distance `200`.

Docs: <https://manual.mikrotik.com/docs/user-guides/routing-and-networking-protocols/vrf> ; <https://manual.mikrotik.com/docs/cli-reference/ip/vrf>
