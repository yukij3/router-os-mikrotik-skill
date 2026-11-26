# OpenVPN Details

Referenced from `routeros-vpn` SKILL.md.

## Server and Client

- `/interface/ovpn-server/server` (7.17+: multiple entries with `name`, each with `vrf=`), `/interface/ovpn-client`.
- `protocol=tcp|udp` (default `tcp`), port `1194`, IPv4 and IPv6.
- `mode=ip` (tun) or `mode=ethernet` (tap).
- Server `user-auth-method=mschap2|pap` (7.17+). Passwords up to 1000 characters (7.18+).

## Ciphers and Auth

| Setting | Values |
|---|---|
| `cipher` | `aes128-cbc`, `aes192-cbc`, `aes256-cbc`, `aes128-gcm`, `aes192-gcm`, `aes256-gcm`, `blowfish128`, `null` |
| `auth` | `md5`, `sha1`, `sha256`, `sha384` (7.20+), `sha512`, `null` |

Defaults: server `cipher=aes128-cbc,blowfish128`, client `blowfish128` with `sha1`. Set `cipher=aes256-gcm auth=sha256` (or `auth=null` with GCM).

No LZO compression, no NCP negotiation: the client `.ovpn` must specify the cipher.

## Route Pushing

- Server `push-routes="<net> <mask> <gw>, ..."` (7.14+; max 1400 characters / 37 routes).
- Server `push-routes-ipv6=` (7.21+).
- Or `redirect-gateway=def1|ipv6`.
- Client `route-nopull=yes` ignores pushed routes.

## `.ovpn` Export and Import

Export a client profile from the server (requires `require-client-certificate=yes` and, since 7.18, a server `name`):

```routeros
/interface/ovpn-server/server/export-client-configuration server=ovpn-server1 \
  server-address=<public-address> ca-certificate=<ca> \
  client-certificate=<client-cert> client-cert-key=<key>
```

Import a profile on a RouterOS client:

```routeros
/interface/ovpn-client/import-ovpn-configuration file-name=client.ovpn \
  ovpn-user=<user> ovpn-password=<password> key-passphrase=<passphrase>
```

TLS keys are supported on the **client only**, via `.ovpn` import: `tls-auth` (7.12+), `tls-crypt` and `tls-crypt-v2` (7.17+). With `tls-auth` the client file uses `auth SHA256` and `key-direction 1`.

Docs: <https://manual.mikrotik.com/docs/virtual-private-networks/openvpn>
