---
name: routeros-fundamentals
description: "RouterOS v7 domain knowledge for AI agents: what RouterOS is (and is not), CLI path syntax, REST and binary API rules, /ip/service, users and policies, versions and upgrade channels, packages, device-mode, licensing, and renamed/removed menus. Use when: working with MikroTik RouterOS, writing RouterOS CLI/script commands, calling the RouterOS REST API or API socket, debugging why a Linux command fails on RouterOS, installing packages, changing device-mode, or when the user mentions MikroTik, RouterOS, CHR, WinBox, /rest, or /ip /system /interface paths. Scope: RouterOS 7.x only — v6 is NOT covered and accuracy for v6 problems will be low."
---

# RouterOS Fundamentals

> Verified against RouterOS **7.25.2 stable** and **7.24.7 long-term**, checked 2026-11-26.
> Features newer than 7.12 are tagged with the version that introduced them.

## RouterOS Is NOT GNU/Linux

RouterOS 7.x runs a Linux kernel (still **5.6.3**; v6 used 3.3.5), but **everything above the kernel is MikroTik's proprietary system**. This is the single most important fact for agents to internalize.

**What does NOT exist on RouterOS:**
- No `/bin`, `/usr`, `/etc`, `/var` — no FHS layout
- No bash, sh, ash, zsh — no Unix shell at all
- No coreutils (`ls`, `cat`, `grep`, `ps`, `mount`, `ip`, `iptables`, etc.)
- No glibc, musl, busybox
- No apt, pkg, opkg — packages are `.npk` files managed by `/system/package`
- No `systemctl`, `service`, init system
- No `/proc` or `/sys` accessible from userland
- No `docker`, `podman` — RouterOS has its own `/container` subsystem

**What DOES exist:**
- RouterOS CLI — its own language, not shell. Accessed via SSH, serial, telnet, WinBox, or WebFig
- REST API at `/rest/` (served by the `www` / `www-ssl` services)
- Binary API on `8728/tcp` (`api`) and `8729/tcp` (`api-ssl`)
- RouterOS scripting language (`.rsc` files) — see the `routeros-scripting` skill
- WebFig (web UI) and the WinBox protocol on `8291/tcp`

**Common agent mistakes to avoid:**
- Do NOT try `ssh admin@host 'ls /'` — it opens the RouterOS CLI, not a shell
- Do NOT suggest `mount`, `fdisk`, `mkfs` — use `/disk` commands instead
- Do NOT look for config files at `/etc/` — configuration is in the RouterOS database
- Do NOT assume `ping` works the same — it's `/tool/ping` or `/ping` in CLI
- Do NOT suggest `apt`/`opkg`. Extra packages are fetched with `/system/package/update/check-for-updates` + `/system/package/enable <pkg>` + `/system/package/apply-changes` (7.18+), or uploaded as `.npk` and installed on reboot. See [Extra packages](./references/extra-packages.md)

## RouterOS CLI Syntax

RouterOS CLI uses path-based navigation, not Unix command pipelines:

```routeros
# Navigation (path elements separated by "/" or spaces)
/ip/address/print
/ip address print
/system/resource/print

# Relative navigation: ".." moves up one level, "/" goes to root
/ip/firewall/filter
../service-port/print

# Adding entries
/ip/address/add address=192.168.1.1/24 interface=ether1

# Modifying (by internal ID or find expression)
/ip/address/set [find interface=ether1] address=10.0.0.1/24

# Removing
/ip/address/remove [find address="192.168.1.1/24"]

# Running a command
/system/reboot
/tool/fetch url="http://example.com/file.npk" dst-path="/"
```

**Key syntax differences from shell:**
- `=` assigns properties (no spaces around it)
- `[find ...]` is the query expression (like WHERE)
- Interactive `print` row numbers are not script-safe object IDs; use `[find ...]` or `*HEX` internal IDs
- Strings use `""` (double quotes only)
- Comments use `#`
- Variables: `:local myVar "value"` and `$myVar`
- No pipes, no redirection, no subshell

**Recent console additions** (check the router version before relying on them):

| Version | Addition |
|---|---|
| 7.15 | `/console/settings sanitize-names` (default `no`) — replaces `\ / : * ? \|` with `_` in file/script names |
| 7.16 | `:lock` locks the console session; `import ... verbose=yes dry-run` simulates an import; `verbose=progress` |
| 7.17 | `print group-by=` |
| 7.18 | Scriptable safe mode; `/console/settings log-script-errors` (default `yes`) |
| 7.20 | `/console/settings tab-width` (default `4`) |
| 7.21 | Sensitive values hidden in `print` unless `show-sensitive` is passed (previously only `export` hid them) |
| 7.22 | `print comments` / `!comments`; `print follow` / `follow-only` accept `on-event` and timestamps |
| 7.23 | `export path=`; passing a non-existent parameter is now a runtime error |
| 7.24 | `print order-by=` (up to three properties, `-prop` for descending) |

```routeros
/interface/print order-by=-link-downs
/ip/address/print !comments
/export path=cfg/backup.rsc
```

v7 moved routing configuration: routing tables live in `/routing/table`, policy rules in `/routing/rule`, BGP in `/routing/bgp/{connection,template,session}`, OSPF v2+v3 in `/routing/ospf`, filters in `/routing/filter/rule`. See the `routeros-routing` skill.

## REST API

RouterOS REST API at `http(s)://HOST/rest/`. HTTP verbs map non-standardly:

| HTTP | RouterOS Action | CLI Equiv | Success response |
|---|---|---|---|
| `GET` | print (list/read) | `/path/print` | array, or object for `/*ID` and singletons |
| `PUT` | **add (create)** | `/path/add` | `201 Created` + full created record + `Location` header |
| `PATCH` | set (update) | `/path/set` | updated record |
| `DELETE` | remove | `/path/remove` | empty body |
| `POST` | command (execute) | `/path/<command>` | `POST .../add` returns `200` + `{"ret":"*ID"}` |

**Key gotchas:**
- `PUT` creates (NOT updates) — opposite of many REST APIs
- REST users need **both** `rest-api` **and** `api` policies; missing `api` gives `500, std failure: not allowed (9)`
- Hard **60-second** limit per request → `400 {"detail":"Session closed"}`; `duration=` longer than 60s does not extend it
- Requests from the **same user** run serially; different users run in parallel. A stuck request stalls that user's queue — use separate users for parallel clients
- Do NOT percent-encode `*` in IDs (`/rest/ip/address/*1`, not `%2A1`). Other escapes (`%20`, `%2F`) are decoded since 7.21
- Singleton menus (`/system/identity`, `/ip/dns`, `/ip/service/webserver`) reject `PATCH`; use `POST /rest/<menu>/set`
- Ordered menus support `POST .../move` with `{".id":"*9","destination":"*C"}`
- No CORS headers; `OPTIONS` preflight returns `503` — browser apps need a proxy
- HTTP Basic auth only. Empty password: `admin:` (colon required, nothing after)
- WebFig root (`GET /`) returns HTTP 200 without auth — usable as a health check; `/rest/` needs auth (401 without)
- `.id` values are `*HEX` (e.g., `*1`, `*A`); all values are strings
- 7.21+: `/ip/service/webserver` toggles index/webfig/graphs/rest separately for HTTP (`*-plain`) and HTTPS (`*-secure`); disabled parts return `403`

See [REST API reference](./references/rest-api-patterns.md) for print/query/proplist, the binary API, and `/console/inspect`, and [Async commands](./references/async-commands-rest.md) for monitor/streaming commands.

## Services (`/ip/service`)

| Service | Port | Notes |
|---|---|---|
| `api` | `8728/tcp` | Binary API |
| `api-ssl` | `8729/tcp` | Needs a real `certificate`; `none` means anonymous DH ciphers that modern TLS clients refuse |
| `www` / `www-ssl` | 80 / 443 | WebFig + REST (same certificate caveat for `www-ssl`) |
| `winbox` | `8291/tcp` | |
| `reverse-proxy` | `443/tcp` | Listens only when an enabled rule exists |

- 7.24 renamed `address` → **`available-from`** (`address` still accepted as a deprecated alias). It still completes the TCP handshake before closing, so keep input firewall rules.
- Other parameters: `tls-version` (`any | only-1.2 | only-1.3 | min-1.2 | min-1.3`, default `any`), `vrf` (default `main`), `max-sessions` (`1..1000`, default `20`).
- `/ip/service/print where dynamic` also lists dynamic listeners and connections.

```routeros
/ip/service/set api-ssl certificate=my-cert tls-version=min-1.2 available-from=10.0.0.0/8
/ip/service/webserver/set rest-plain=no
```

## Version Scheme and Channels

Format: `MAJOR.MINOR[.PATCH][betaN|rcN]` — e.g., `7.25.2`, `7.24.7`, `7.26beta7`. Components are compared as integers (`7.25 > 7.9`); `betaN` < `rcN` < release.

| Channel | Current (2026-11-26) |
|---|---|
| `stable` | `7.25.2` |
| `long-term` | `7.24.7` |
| `testing` | `7.25.2` |
| `development` | `7.26beta7` |

- Version endpoint (plain text): `https://upgrade.mikrotik.com/routeros/NEWESTa7.<channel>`
- Changelog: `https://download.mikrotik.com/routeros/<version>/CHANGELOG`
- Select the channel with `/system/package/update/set channel=<channel>`
- 7.25 dropped the channel label from the version string — do not parse `(stable)` out of it

See [Version parsing reference](./references/version-parsing.md) for comparison code and download URLs.

## Architecture Names

MikroTik uses its own architecture identifiers. Read the router's value from `/system/resource/print` (`architecture-name`).

| MikroTik name | CPU |
|---|---|
| `x86` | x86_64 (also used for CHR) |
| `arm64` | aarch64 |
| `arm` | ARMv7 |
| `mipsbe` | MIPS big-endian |
| `mmips` | MIPS multi-core |
| `smips` | MIPS single-core |
| `ppc` | PowerPC |
| `tile` | Tilera |

CHR (Cloud Hosted Router) is available only for `x86` and `arm64`. Extra-package availability differs per architecture — see [Extra packages](./references/extra-packages.md).

## Default Credentials and First Login

- Username: `admin`
- Password: newer MikroTik hardware ships with a **unique factory password** printed on the serial-number sticker / Quick Guide / device compartment. Older devices and CHR start with an empty password.
- When the password is expired, interactive login prompts `Change your password (Ctrl-C to skip)` (wording since 7.21). There is no `a` key to skip.
- REST and API sockets are not blocked by the expired-password flag.

## Users and Policies (Summary)

- Default groups `read`, `write`, `full` cannot be deleted. `read` already includes `sensitive`, `reboot`, `sniff`, `api`, `rest-api` — do not hand it to untrusted users.
- 17 policies: `local telnet ssh ftp reboot read write policy test winbox password web sniff sensitive api romon rest-api`.
- `/user/group/add` revokes every policy not listed; `/user/group/set` changes only listed ones (`!policy` revokes).
- `/user/add` needs `name`, `group` and `password` (pass `password=""` explicitly for an empty one).
- 7.16: `inactivity-timeout` (default `10m`) and `inactivity-policy` (`none | logout | lockscreen`, default `none`).
- 7.20: `/user/active/request-logout` → `/user/active/remove`.
- SSH: 7.21 added ED25519-SK keys, renamed ssh-key `key-owner` → `info`, replaced `/ip/ssh always-allow-password-login` with `password-authentication`; 7.24 added `mlkem768x25519-sha256` KEX; 7.25 made Ed25519 the default host key and added host-key verification to the SSH client.

Details and REST shapes: [Users REST reference](./references/routeros-users-rest.md).

## Packages, Device-Mode, Licensing (Summary)

- **Packages:** `/system/package/apply-changes` (7.18+) commits enable/disable/uninstall and reboots; a plain `/system/reboot` **discards** those pending changes on 7.18+. Uploaded `.npk` files install on reboot; a file named `<pkg>.auto.npk` uploaded via FTP/SFTP triggers reboot + install automatically. → [Extra packages](./references/extra-packages.md), [Packages REST](./references/packages-rest.md)
- **Device-mode:** modes `home`, `basic`, `advanced` (formerly `enterprise`, 7.17), `rose` (7.19). Changing it needs a cold power cycle or button press within `activation-timeout`; `mode=` wipes per-feature overrides not repeated in the same command. → [Device-mode](./references/device-mode.md), [Device-mode REST](./references/device-mode-rest.md)
- **Licensing:** RouterBOARD/x86 use levels 0–6; CHR uses `free` (1 Mbps upload per interface), `p1`, `p10`, `p-unlimited`. → [Licensing REST](./references/licensing-rest.md)

## Renamed and Removed (7.x)

| Old | New / status | Since |
|---|---|---|
| device-mode `enterprise` | `advanced` | 7.17 |
| `/system/upgrade` | `/system/package/local-update` | 7.17 |
| `/ip/ssh allow-none-crypto` | `ciphers` | 7.17 |
| `print follow-strict` | removed | 7.16 |
| `/system/resource/usb`, `/system/resource/pci` | `/system/resource/hardware` (disks added 7.21) | 7.20 |
| `/user/active/request-logout` | `/user/active/remove` | 7.20 |
| `/quickset` | removed | 7.21 |
| `www` publishing directories via `/files` | removed | 7.21 |
| `/ip/ssh always-allow-password-login` | `password-authentication` | 7.21 |
| `/user/ssh-keys key-owner` | `info` | 7.21 |
| `/ip/service address` | `available-from` (alias kept) | 7.24 |
| `factory-software` / `factory-firmware` | `minimum-version` / `minimum-firmware` | 7.24 |
| API MD5 challenge-response login | rejected — send `=name=` + `=password=` | current |

## Inspecting Hardware from RouterOS CLI

```routeros
# PCI/USB devices and disks (7.20+; replaces /system/resource/pci and /usb)
/system/resource/hardware/print

# IRQ assignments (shows driver binding)
/system/resource/irq/print

# System overview (version, architecture-name, board-name, CPU, memory)
/system/resource/print

# Disk info
/disk/print

# Installed packages
/system/package/print

# IP services and ports
/ip/service/print

# Network interfaces
/interface/print
```

## Additional Resources

**Official docs** (manual.mikrotik.com replaces the older Confluence-based `help.mikrotik.com`):
- Manual home: <https://manual.mikrotik.com/>
- CLI Reference: <https://manual.mikrotik.com/docs/cli-reference/>
- REST API: <https://manual.mikrotik.com/docs/developer-guides/rest-api/>
- Binary API: <https://manual.mikrotik.com/docs/developer-guides/api/>
- Console: <https://manual.mikrotik.com/docs/management-tools/console/>
- Services: <https://manual.mikrotik.com/docs/system-information-and-utilities/services/>
- Packages: <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/packages/>
- Device-mode: <https://manual.mikrotik.com/docs/system-information-and-utilities/device-mode/>
- Users: <https://manual.mikrotik.com/docs/authentication-authorization-accounting/user/>

**Reference files:**
- REST API patterns, binary API, `/console/inspect`: [rest-api-patterns.md](./references/rest-api-patterns.md)
- Monitor/streaming commands over REST: [async-commands-rest.md](./references/async-commands-rest.md)
- Version parsing, comparison, download URLs: [version-parsing.md](./references/version-parsing.md)
- Extra packages per architecture: [extra-packages.md](./references/extra-packages.md)
- `/system/package` over REST: [packages-rest.md](./references/packages-rest.md)
- Device-mode modes and feature matrix: [device-mode.md](./references/device-mode.md); REST behavior: [device-mode-rest.md](./references/device-mode-rest.md)
- Licensing levels and `/system/license` REST: [licensing-rest.md](./references/licensing-rest.md)
- Users, groups, SSH keys: [routeros-users-rest.md](./references/routeros-users-rest.md)
- IP addressing, routes, DHCP client, DNS, pools: [routeros-networking-rest.md](./references/routeros-networking-rest.md)
- Firewall filter/NAT/mangle over REST: [routeros-firewall-rest.md](./references/routeros-firewall-rest.md)
- Bun runtime bugs affecting HTTP clients: [bun-runtime-gotchas.md](./references/bun-runtime-gotchas.md)

**Related skills:**
- Scripting and `.rsc`/CLI config automation: `routeros-scripting`
- `/console/inspect` tree traversal and schema generation: `routeros-command-tree`
- Firewall design (filter, NAT, address-lists): `routeros-firewall`
- Bridges and VLANs: `routeros-bridge-vlan`
- Wi-Fi and the wireless driver packages: `routeros-wifi`
- Routing (`/routing/table`, BGP, OSPF, routing filters): `routeros-routing`
- VPN tunnels: `routeros-vpn`
- Upgrades, backups and other maintenance: `routeros-maintenance`
- `/container` subsystem: `routeros-container`; `/app` YAML (7.22+): `routeros-app-yaml`
- Netinstall and device flashing: `routeros-netinstall`
- CHR in QEMU (local or CI): `routeros-qemu-chr`
- Packet capture and TZSP: `routeros-sniffer`

**MCP tools:**
- For command tree browsing and property lookups: use the `rosetta` MCP server tools (`routeros_search`, `routeros_get_page`, `routeros_command_tree`) when available
