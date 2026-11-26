# WireGuard Details

Referenced from `routeros-vpn` SKILL.md.

## Interface (`/interface/wireguard`)

| Property | Notes |
|---|---|
| `name`, `listen-port` | Port must be unique per interface (7.14+); `listen-port=0` picks one automatically (7.24+) |
| `private-key`, `public-key` | Generated on add |
| `vrf` | 7.21+, CLI only: VRF of the **encrypted UDP socket**, not of the tunnel interface |

## Peers (`/interface/wireguard/peers`)

| Property | Notes |
|---|---|
| `interface` | WireGuard interface |
| `public-key` | Peer's public key |
| `allowed-address` | Mandatory. Must not overlap with other peers on the same interface (7.24+ warns) |
| `endpoint-address`, `endpoint-port` | Where to send handshakes (initiator side) |
| `persistent-keepalive` | Keeps NAT mappings open |
| `preshared-key` | `auto` / `none` accepted (7.12+) |
| `private-key` | `auto` / `none` accepted (7.12+) |
| `name` | 7.15+; appears in logs, usable as a peer reference (7.25+) |
| `responder` | 7.15+; `yes` = never initiate handshakes |

### Client config properties

| Property | Notes |
|---|---|
| `client-address` | Address put into the generated config; default `192.168.177.2/24` when empty |
| `client-dns` | Domain names accepted since 7.24 |
| `client-endpoint` | Public address clients connect to |
| `client-keepalive`, `client-listen-port` | |
| `client-allowed-address` | 7.21+ (previously hardcoded to `0.0.0.0/0, ::/0`) |
| `client-mtu` | 7.25+ |

```routeros
/interface/wireguard/peers/show-client-config [find name=laptop]
/interface/wireguard/peers/show-client-config [find name=laptop] file=laptop
```

Since 7.21 `show-client-config` needs the `show-sensitive` user policy.

> Default `client-allowed-address` is `0.0.0.0/0, ::/0` (WireGuard guide); set it explicitly for split tunnels.

## Import / Export

- `/interface/wireguard/wg-export` (7.12+): standard `[Interface]` / `[Peer]` `.conf`. Keys and peers only, no addresses, routes or firewall.
- `/interface/wireguard/wg-import`:
  - `config-file=<file>` (7.12+)
  - `config-string="..."` (7.19+)
  - Ignores `#` comments and `wg-quick` keys (7.24/7.25+)

## Multi-WAN Server

Replies must leave through the WAN the handshake arrived on:

- Set `responder=yes` on the peers.
- In mangle, mark connections and routing on the `output` chain matching `src-port=13231` (the listen port) per WAN, and route each mark through its WAN table (see `routeros-firewall` for mangle and `routeros-routing` for tables).
- Masquerade in `srcnat` on each WAN.

## HotSpot 2FA over WireGuard (7.22+)

HotSpot can bind to a WireGuard interface (`/ip/hotspot/add interface=wg1`) to add a TOTP login (`otp-secret`) in front of the tunnel. HotSpot setup itself: `routeros-hotspot`.

Docs: <https://manual.mikrotik.com/docs/virtual-private-networks/wireguard> ; <https://manual.mikrotik.com/docs/cli-reference/interface/wireguard/peers/> ; <https://manual.mikrotik.com/docs/cli-reference/interface/wireguard/wg-import> ; <https://manual.mikrotik.com/docs/cli-reference/interface/wireguard/wg-export>
