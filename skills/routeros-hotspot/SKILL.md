---
name: routeros-hotspot
description: "RouterOS v7 hotspot captive portal for wired/wireless access control. Use when: configuring hotspot on RouterOS, setting up a captive portal, writing hotspot profiles or instances, configuring walled garden, the RFC 8910 Captive Portal API (DHCP option 114, api.json), integrating RADIUS, RadSec or User Manager with hotspot, hotspot TOTP (otp-secret), enabling hotspot under device-mode, hotspot with multi-WAN/policy routing, or when the user mentions /ip/hotspot, walled-garden, hotspot profile, login.html, or captive portal on MikroTik."
---

# RouterOS Hotspot

Verified against RouterOS **7.25.2** stable / **7.24.7** long-term (checked 2026-11-26). Features newer than 7.12 are tagged with their minimum version.

## Before You Start: device-mode and Packages

- **device-mode (7.17+ factory default):** routers ship in `home` mode (home routers), `basic` (most others) or `advanced` (CCR, 1100 series). In `home` and `basic`, **`hotspot` and `proxy` are disabled** and `/ip/hotspot` configuration fails. Enable it, then confirm physically (button press or cold power cycle within 5 minutes):

  ```routeros
  /system/device-mode/update hotspot=yes
  ```

- **smips devices (7.20+):** hotspot is no longer in the base package; install the separate `hotspot` package.

## How Hotspot Chains Work

Hotspot traffic intercept runs **before** the regular firewall input/forward chains. This is the single most important fact to internalize:

- `/ip/hotspot` binds to a bridge or interface — all traffic on that interface enters the hotspot chain first
- Firewall rules blocking TCP 80/443 from the hotspot interface do **NOT** block the captive portal login page — hotspot handles it before the firewall sees it
- RouterOS automatically injects dynamic firewall rules (`hs-unauth`, `hs-auth` chains) — do not manually create, remove, or interfere with these hotspot-managed rules
- Firewall rules can target hotspot traffic with the `hotspot=` matcher: `auth`, `from-client`, `to-client`, `local-dst`, `http` (see `routeros-firewall`)

**Common mistake:** Adding a DROP rule for port 443 from bridge-hotspot to "fix a security gap" — this breaks the HTTPS login page silently.

**IPv4-only:** hotspot relies on NAT and the IPv4 proxy. IPv6 clients are not supported.

## Hotspot Profile

```routeros
/ip/hotspot/profile/add \
  name=my-profile \
  hotspot-address=10.20.0.1 \
  login-by=https,mac,http-pap \
  mac-auth-mode=mac-as-username-and-password \
  dns-name=login.example.com \
  ssl-certificate=login.example.com.crt_0 \
  nas-port-type=ethernet \
  use-radius=yes \
  radius-accounting=yes \
  html-directory-override=hotspot-files
```

Key properties:
- `ssl-certificate=` — reference the name after import (RouterOS appends `_0` to imported certificate names)
- `nas-port-type=` — `ethernet`, `cable` or `wireless-802.11` (default `wireless-802.11`). Use `ethernet` for wired hotspots
- `html-directory-override=` — must match the exact folder name on the router's filesystem (`flash/` prefix handling was fixed in 7.17.2/7.18 and again in 7.25.2)
- `login-by=https` — serves the login page over HTTPS; requires `www-ssl` service enabled with the same certificate
- `dns-name=` + a valid `ssl-certificate=` — also enable the automatic Captive Portal API (see below)
- `use-radius=yes` — RADIUS is consulted **only if no matching local `/ip/hotspot/user` exists**. Local users are checked first
- `mac-auth-password` is marked `sensitive` since 7.24 (hidden from non-sensitive exports)

### Local Users Win Over RADIUS

With `use-radius=yes`, RouterOS checks `/ip/hotspot/user` **first** and queries RADIUS only when no local user matches. The `admin` user that `/ip/hotspot/setup` creates will authenticate locally and never reach RADIUS. Remove or disable local users you do not want, when RADIUS is meant to be authoritative.

## Hotspot Instance

```routeros
/ip/hotspot/add \
  name=hotspot1 \
  interface=bridge-hotspot \
  profile=my-profile \
  address-pool=pool-hotspot \
  addresses-per-mac=2 \
  idle-timeout=5m \
  keepalive-timeout=none \
  disabled=no
```

**Note:** `keepalive-timeout=none` disables the keepalive. `keepalive-timeout=0` is NOT valid — it is ignored.

## Captive Portal API (RFC 8910 / DHCP Option 114)

**Since 7.3, hotspot sends the Captive Portal DHCP option automatically** (RFC 7710 / RFC 8910), pointing to `https://<dns-name>/api`, whenever the hotspot profile has `dns-name` and a valid `ssl-certificate`. No `/ip/dhcp-server/option` is needed for the normal case.

The API response is `api.json` in the hotspot HTML directory (visible under Files). The official template includes `"can-extend-session": true` and uses `$(remain-bytes-total)`. If the router was upgraded from a version before 7.3, use `/ip/hotspot/reset-html` to get the current template files, including `api.json`. When you use `html-directory-override`, make sure your custom folder also contains an `api.json`.

> Caveat: the docs do not say when RouterOS (re)creates `api.json` relative to `html-directory-override`. Check the Files list on the target device instead of assuming.

### Manual Option 114 (override only)

Only configure option 114 by hand to point clients at a different URL or to serve it outside built-in hotspot handling:

```routeros
/ip/dhcp-server/option/add \
  name=captive-portal \
  code=114 \
  force=yes \
  value="'https://login.example.com/api'"

/ip/dhcp-server/option/sets/add \
  name=captive-portal-set \
  options=captive-portal

/ip/dhcp-server/set my-dhcp-server dhcp-option-set=captive-portal-set
```

- **Option value syntax:** outer double quotes, inner single quotes — `"'https://...'"`. Missing inner quotes send the option as a binary blob, not a string URI.
- `force=yes` sends the option even when the client's Parameter Request List does not ask for code 114.

> Caveat: the docs do not detail how the automatic option interacts with a manually configured code 114 option (which one wins, whether the automatic one is forced). Test on the target device if you configure both.

## Walled Garden

Use a consistent `comment=` tag for idempotent add/remove. Without it, repeated script runs accumulate duplicate entries.

```routeros
# Remove only our entries, not manually-added ones
/ip/hotspot/walled-garden/ip/remove [find comment="my-wg"]

/ip/hotspot/walled-garden/ip/add dst-host=example.com   action=accept comment="my-wg"
/ip/hotspot/walled-garden/ip/add dst-host=*.example.com action=accept comment="my-wg"
```

**IP vs HTTP walled garden:**
- `/ip/hotspot/walled-garden/ip` — layer 3 match by destination host, applied BEFORE authentication. Use for HTTPS destinations.
- `/ip/hotspot/walled-garden` — layer 7 URL pattern match, requires HTTP. Does NOT work for HTTPS.
- Action enums differ: `walled-garden` uses `action=allow|deny` (default `allow`); `walled-garden/ip` uses `action=accept|drop|reject`.
- In `walled-garden/ip`, `dst-address` and `dst-address-list` are **ignored** when `dst-host` is set (the host is resolved dynamically).

## SSL Certificate Note

The hotspot profile references `ssl-certificate=name.crt_0` (RouterOS appends `_0` on import). Enable `www-ssl` with the same certificate:

```routeros
/ip/service/set www-ssl disabled=no certificate=login.example.com.crt_0 tls-version=only-1.2
```

In 7.25 `tls-version` is `any | only-1.2` (default `any` = TLS 1.0, 1.1 and 1.2 — no TLS 1.3). TLS 1.3 options (`only-1.3`, `min-1.2`, `min-1.3`) appear in 7.26.

Quick import pattern:
```routeros
/certificate import file-name=login.example.com.crt passphrase=""
/certificate import file-name=login.example.com.key passphrase=""
# After import, the certificate appears as login.example.com.crt_0
```

## Hotspot with Multi-WAN / Policy Routing

By default, mangle routing marks are evaluated before the `local` destination lookup. With PCC or other mangle-based policy routing on hotspot traffic, this breaks portal interception (HTTP 80 detection, login page). Fix it in the routing policy, not by excluding hotspot from the firewall:

- **7.22+:** reorder `/routing/settings policy-rules` so `local` is evaluated before `mangle`.
- Any v7: add `/routing/rule` entries that send local hotspot traffic to the right table before the marked traffic.

Exact syntax for both is in the `routeros-routing` skill. `/ip route rule` is v6 syntax, and the transitional v7 menu `/routing/route/rule` was removed in 7.21.5 / 7.23.1 — use `/routing/rule`.

FastTrack also bypasses the hotspot universal client; see `routeros-firewall`.

## Hotspot TOTP (7.21+)

Local `/ip/hotspot/user` entries support `otp-secret` for TOTP:
- Added as `totp-secret` in 7.21, **renamed to `otp-secret` in 7.21.3** — scripts written for 7.21.0–7.21.2 must be updated.
- Since 7.22 the value is validated as base32 and marked `sensitive`.

## Security Hardening Notes

- `dst=` login parameter only accepts `http:` and `https:` schemes (7.19.2+); reserved URI characters are escaped (7.16+).
- `on-login` / `on-logout` scripts run with restricted permissions since 7.12: `read,write,reboot,ftp,romon,test` only (no `policy`, `sensitive`, `password`). Scripts needing more will fail.
- There is no HTML-escape template variant — see the template section below.

## External Captive Portal — HTML Template Variables

RouterOS substitutes `$(variable)` server-side in any file under `html-directory-override` before serving it. These are **not** JavaScript variables — they are filled before the browser receives the page.

Most external captive portal setups override `login.html`, `alogin.html`, and `status.html`. Keep the full servlet page list, POST fields, RADIUS pass-through variables, multi-language `target=` mechanics, and HTTP response-control syntax in [references/template-variables.md](./references/template-variables.md).

Use `$(link-login-only)` as the login POST endpoint, pass the original destination through `dst`, and always pick escaped variants such as `$(link-orig-esc)`, `$(mac-esc)`, and `$(username-esc)` when embedding values into URLs or query strings. The non-escaped variants can break parsing or open injection paths if a value contains `&`, `=`, `?`, or `#`.

`-esc` is URL-encoding only. There is **no HTML-escape variant**: never place user-controlled values (`$(username)`, `$(error-orig)`) unescaped inside HTML attributes or `<script>` blocks.

**Conditional syntax** (server-side, not JavaScript):

```
$(if logged-in == "yes")
  <a href="$(link-logout)">Logout</a>
$(else)
  <form action="$(link-login-only)" method="post">...</form>
$(endif)
```

Operators: presence (`$(if VAR)`), `==`, `!=`. Also `$(elif ...)`. No nested arithmetic / string ops.

Generic `login.html` pattern for external captive portal:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta http-equiv="pragma" content="no-cache">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <!-- External auth provider loads redirect logic -->
  <script src='https://auth.example.com/snippets/router-os/redirect'></script>
</head>
<body>
  <div id="entryPoint">Redirecting...</div>
  <script>
    window.addEventListener('load', function () {
      ExternalAuth_redirect(
        '$(server-name)', '$(mac-esc)', '$(link-login-only)',
        '$(link-orig-esc)', '$(error-orig)', '$(logged-in)', '$(ip)'
      );
    });
  </script>
</body>
</html>
```

The external provider authenticates the user and redirects the browser back to `$(link-login-only)` with `username` + `password` + `dst` POST fields. RouterOS validates (local users first, then RADIUS), issues the auth cookie, and redirects to `dst`.

**Walled-garden must include the external auth host (and any CDN/asset hosts)** — otherwise the unauthenticated browser cannot reach the redirect script.

## Common LLM Mistakes

| Mistake | Correct behavior |
|---------|-----------------|
| DROP TCP 443 from hotspot interface | Hotspot chain runs before firewall — breaks the HTTPS login page |
| `keepalive-timeout=0` to disable keepalive | Use `keepalive-timeout=none` |
| `nas-port-type=wireless-ieee-802-11-g` | Valid values: `ethernet`, `cable`, `wireless-802.11` |
| Assuming `use-radius=yes` bypasses local users | Local `/ip/hotspot/user` is checked first; RADIUS only if no local match |
| Leaving the setup-created `admin` hotspot user with RADIUS | It authenticates locally and never hits RADIUS |
| Manually adding DHCP option 114 for every hotspot | Automatic since 7.3 when `dns-name` + valid `ssl-certificate` are set |
| Option 114 `value=` without inner single quotes | Must be `"'https://...'"` — outer double, inner single |
| `/ip/hotspot/add` fails on a new router | Check `/system/device-mode` — hotspot is off in `home`/`basic` (7.17+) |
| `totp-secret=` on 7.21.3+ | Renamed to `otp-secret` |
| Wildcard HTTPS domains in `/ip/hotspot/walled-garden` | Use `/ip/hotspot/walled-garden/ip` for HTTPS (layer 3 match) |
| Hotspot on dual-stack network with IPv6 | Hotspot is IPv4-only — IPv6 not supported |
| `$(link-login)` treated as JavaScript variable | It is RouterOS server-side substitution, not JS |
| Embedding `$(link-orig)` / `$(mac)` / `$(username)` into another URL | Use the `-esc` variant — non-esc breaks parsing on `&`, `=`, `?` and may enable injection |
| `/ip route rule` for hotspot on multi-WAN | v6 syntax. Use `/routing/rule`, or `/routing/settings policy-rules` on 7.22+ |
| `on-login` script that needs `policy`/`sensitive` | Hotspot scripts are limited to `read,write,reboot,ftp,romon,test` (7.12+) |

## RADIUS Integration

See [references/radius-client.md](./references/radius-client.md) for RADIUS client configuration, the 7.15+ `require-message-auth` default, RadSec, CoA/Disconnect and User Manager.

## Additional Resources

**Related skills:**
- `routeros-fundamentals` — RouterOS CLI syntax, REST API, scripting basics
- `routeros-firewall` — `hotspot=` matcher, mangle, FastTrack
- `routeros-routing` — `/routing/rule`, `/routing/settings policy-rules`
- `routeros-certificates` (in backlog) — for the full certificate chain handling pattern

**MCP tools:**
- `rosetta` MCP server — `/tool/ping`, `/ip/hotspot` command tree inspection (`routeros_search`, `routeros_get_page`)

**MikroTik docs:**
- [HotSpot - Captive portal](https://manual.mikrotik.com/docs/authentication-authorization-accounting/hotspot-captive-portal/)
- [Hotspot customisation](https://manual.mikrotik.com/docs/authentication-authorization-accounting/hotspot-captive-portal/hotspot-customisation)
- [/ip/hotspot/profile (CLI)](https://manual.mikrotik.com/docs/cli-reference/ip/hotspot/profile)
- [RADIUS](https://manual.mikrotik.com/docs/authentication-authorization-accounting/radius)
- [Device-mode](https://manual.mikrotik.com/docs/system-information-and-utilities/device-mode)
- [DHCP](https://manual.mikrotik.com/docs/network-management/dhcp/) — manual option configuration
