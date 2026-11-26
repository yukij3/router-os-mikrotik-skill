# RADIUS Client Configuration

RouterOS RADIUS client for hotspot authentication. Referenced from `routeros-hotspot` SKILL.md. Verified against RouterOS 7.25.2 stable / 7.24.7 long-term.

## Lookup Order: Local Users First

With `use-radius=yes` on the hotspot profile, RouterOS checks `/ip/hotspot/user` **first**. RADIUS is consulted only when no matching local user exists. A leftover local user (for example the `admin` user that `/ip/hotspot/setup` creates) authenticates locally and never reaches RADIUS.

## Multi-Server Pattern

RouterOS tries RADIUS servers in list order and falls back to the next on timeout. Add at least two entries for redundancy:

```routeros
/radius add \
  address=198.51.100.10 \
  secret="your-shared-secret" \
  service=hotspot \
  timeout=3s \
  comment="radius-primary"

/radius add \
  address=198.51.100.11 \
  secret="your-shared-secret" \
  service=hotspot \
  timeout=3s \
  comment="radius-secondary"
```

Defaults: `timeout=1100ms` for UDP RADIUS, `radsec-timeout=3300ms` for RadSec. The examples raise `timeout` to `3s` for slow or remote servers; leave it at the default for a local server.

`address=` also accepts `ipv4@vrf` / `ipv6@vrf` to reach a server inside a VRF.

## `require-message-auth` (7.15+, BlastRADIUS)

Since 7.15, the `/radius` client and User Manager put `Message-Authenticator` in all non-accounting RADIUS messages, and `/radius` has:

`/radius` `require-message-auth` = `no | yes-for-request-resp` (default `yes-for-request-resp`). User Manager uses a different enum: `/user-manager` `require-message-auth` = `no | yes-access-request`.

Older FreeRADIUS or third-party servers that do not include `Message-Authenticator` in Access-Accept / Access-Reject / Access-Challenge make authentication **fail silently** after the upgrade. Fix the server; set `require-message-auth=no` only as a temporary workaround.

## RadSec (RADIUS over TLS, RFC 6614)

```routeros
/radius add service=hotspot,ppp address=10.0.0.3 protocol=radsec certificate=client.crt
```

- `protocol=radsec` with a client `certificate=`; timeout is controlled by `radsec-timeout` (default `3300ms`).
- The router connects to TCP **2083**; `secret`, `authentication-port` and `accounting-port` are not used. RFC 6614 fixes the RadSec shared secret to `radsec` — configure that on servers that ask for one.
- Fixes: certificate verification and unresponsiveness (7.19.3), SSL CPU spike on closed connections (7.22). Use at least 7.22 for RadSec.

## Incoming CoA / Disconnect-Message

```routeros
/radius/incoming set accept=yes port=3799 vrf=main
```

Disconnect-Message and CoA (RFC 3576) are supported; PoD is not.

## Initial Disabled State

In automated deployments, create RADIUS entries as `disabled=yes` and enable them only after the hotspot HTML files and certificates are ready:

```routeros
# At deploy time — disabled until hotspot files are ready
/radius add address=198.51.100.10 secret="..." service=hotspot timeout=3s \
  comment="my-radius" disabled=yes

# After hotspot files downloaded:
/radius enable [find comment~"my-radius"]
```

This prevents the hotspot from serving auth requests before the login page exists. Running `enable` again on already-enabled entries has no effect, so the step is idempotent.

## Secret Management

Never hardcode RADIUS secrets in scripts. Use a placeholder that your deployment templating substitutes:

```routeros
# <<radius_secret>> is substituted at deploy time
/radius add address=198.51.100.10 secret="<<radius_secret>>" service=hotspot \
  timeout=3s comment="my-radius" disabled=yes
```

## `[:resolve]` Trap

`[:resolve "radius.example.com"]` returns **only the first A record**. If the RADIUS server has multiple IPs for redundancy, only one is used. Use explicit IP variables:

```routeros
# BAD — only gets 1 IP even if DNS has multiple A records
:local radiusIp [:resolve "radius.example.com"]

# GOOD — use explicit IPs
/radius add address="198.51.100.10" secret="..." service=hotspot comment="radius-1" disabled=yes
/radius add address="198.51.100.11" secret="..." service=hotspot comment="radius-2" disabled=yes
```

## User Manager v7 (Local RADIUS Server)

`/user-manager` is a separate `user-manager` package (all architectures except `smips`). It acts as a RADIUS server for Hotspot, PPP, Wi-Fi/802.1X, DHCP and RouterOS login (2FA with TOTP `otp-secret`).

- RadSec server support since 7.21: `/user-manager/set radsec-certificate=<cert>` and `/user-manager/router/add name=... address=<ip> protocol=radsec`.
- Custom RADIUS attributes, for example `Mikrotik-Group` (maps to a hotspot user profile), `Mikrotik-Rate-Limit`, `Mikrotik-Mark-Id`, `Filter-Id`.
- `use-profiles=yes|no`, vouchers, and CoA (`coa-port=3799`).

## Sources

- [RADIUS](https://manual.mikrotik.com/docs/authentication-authorization-accounting/radius)
- [User Manager](https://manual.mikrotik.com/docs/authentication-authorization-accounting/user-manager)
- [HotSpot - Captive portal](https://manual.mikrotik.com/docs/authentication-authorization-accounting/hotspot-captive-portal/)
