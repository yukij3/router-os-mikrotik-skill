---
name: routeros-maintenance
description: "RouterOS v7 day-2 operations on MikroTik: online/local/manual upgrades and RouterBOARD firmware, binary backup vs /export and /import, cloud backup, reset-configuration, Safe Mode (interactive and scriptable), logging and remote syslog (UDP/TCP/TLS, CEF), NTP client/server, certificates and Let's Encrypt via ACME, built-in CA trust store, IP Cloud DDNS, and the scheduler. Use when: upgrading RouterOS or bootloader firmware, taking or restoring backups, migrating config to another device, resetting a router, avoiding lockout while changing config remotely, sending logs to a syslog server, fixing the clock, issuing an HTTPS certificate, scheduling tasks, or when the user mentions /system/package/update, local-update, .npk, /system/routerboard, /system/backup, /export, /import, .rsc, reset-configuration, safe mode, /system/logging, syslog, /system/ntp, /certificate, add-acme, Let's Encrypt, /ip/cloud, mynetname.net or /system/scheduler."
---

# RouterOS Maintenance (v7)

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26. Features newer than 7.12 carry their minimum version inline.

Version scheme, release channels, packages, `/system/package/enable` + `apply-changes`, and `device-mode` are covered in `routeros-fundamentals`. Script syntax and script/scheduler policies are in `routeros-scripting`.

## Upgrades

### Online (`/system/package/update`)

```routeros
/system/package/update/set channel=stable
/system/package/update/check-for-updates
/system/package/update/install
```

- Channels: `long-term`, `stable`, `testing`, `development`.
- `download` fetches and waits for your reboot; `install` downloads and reboots immediately; `cancel` aborts.
- 7.21+: `check-for-updates` has a `fetch-changelog` flag.
- 7.23+: `mode=https` is the default (`check-certificate=yes`, `ip-version=auto`). A `vrf=` property appears in current docs; check it exists on your version before relying on it.
- 7.18+: `check-for-updates` also lists available extra packages (`XA` flags) in `/system/package`. Enable them with `/system/package/enable <pkg>` then `/system/package/apply-changes`. **`/system/reboot` discards pending package actions.** `apply-changes` no longer needs the `sensitive` policy since 7.24.

### Local mirror (`/system/package/local-update`, 7.17+)

Pulls packages from another RouterOS device or a mirror (`packs/` directory). Replaces the old `/system/upgrade` menu ("system > auto update").

### Manual `.npk`

Upload the `.npk` files to the root of `/file`, then `/system/reboot`. Use `/system/package/update/install ignore-missing` to proceed when some extra packages are missing. Files named `*.auto.npk` uploaded over FTP/SFTP trigger the install and reboot automatically.

### RouterBOARD bootloader firmware

Firmware is separate from RouterOS. After upgrading RouterOS:

```routeros
/system/routerboard/print
# compare current-firmware with upgrade-firmware
/system/routerboard/upgrade
/system/reboot
```

`/system/routerboard/settings/set auto-upgrade=yes` upgrades firmware automatically, but it still needs a second reboot after the OS upgrade. Since 7.17, changing `boot-os` or CPU frequency needs `routerboard=yes` in `device-mode`.

## Backup vs Export

| | `/system/backup` (binary) | `/export` (text `.rsc`) |
|---|---|---|
| Use for | Restoring the **same device**, same RouterOS version | Review, diff, migration to other hardware |
| Contains | Everything incl. users, passwords, certificates, private keys, **interface MACs** | Config only; secrets only with `show-sensitive` |
| Missing | The Dude and User-Manager databases | Users and passwords, certificates, Dude/User-Manager DBs |
| Restore | `/system/backup/load` (reboots) | `/import` (additive, onto existing config) |

```routeros
/system/backup/save name=mybackup password=<secret>
/system/backup/load name=mybackup password=<secret>
/export file=cfg show-sensitive
/import file-name=cfg.rsc verbose=yes dry-run
```

- A backup saved without a password is **unencrypted** (since 6.43). With a password the default encryption is `aes-sha256`; `dont-encrypt=yes` forces an unencrypted backup even when a password is set.
- Restoring a backup onto a different model copies its MAC addresses and is unsafe. Use `/export` + `/import` there.
- For full disaster recovery from `/export`, also export certificates (`/certificate/export-certificate`) and recreate `/user` accounts before import.

Export flags, import behaviour, `*.auto.rsc` and cloud backup: [backup-export.md](./references/backup-export.md).

## Reset Configuration

```routeros
/system/reset-configuration no-defaults=yes
```

Saves `auto-before-reset.backup` (unless `skip-backup=yes`), wipes config, reboots and applies the default config (or the custom Netinstall script).

| Flag | Effect |
|---|---|
| `no-defaults=yes` | Blank config, no default config |
| `keep-users=yes` | Keep user accounts |
| `skip-backup=yes` | No `auto-before-reset.backup` |
| `run-after-reset=file.rsc` | Run a script after reset; **2-minute execution timeout** |
| `caps-mode=yes` | Reset into CAP mode |
| `shutdown=yes` | Shut down after the reset |
| `keep-apps=yes` | Keep apps (7.22+) |

## Safe Mode

Use it for every remote change that could cut your session (firewall, addressing, bridge, VLAN).

| Action | CLI |
|---|---|
| Enter Safe Mode | `Ctrl+X` (or `F4`); prompt shows `<SAFE>` |
| Keep changes and leave | `Ctrl+X` again |
| Undo all changes and leave | `Ctrl+D`, or `/quit` and confirm `y` |
| Connection lost | Changes are undone after the session times out (about 15 min for TCP) |

WinBox and WebFig have a Safe Mode button.

- **100-action limit:** `/system/history` holds 100 actions. Going past it **silently exits Safe Mode and keeps all changes**. Split big changes into smaller Safe Mode sessions.
- Only one session can hold Safe Mode at a time; an unroll undoes Safe Mode changes from other sessions too.
- **Scriptable (7.18+):** `/safe-mode` exposes read-only `enabled`, `user`, `current`, `owner` and the commands `take`, `release`, `unroll`, `toggle`. Their arguments are not yet documented in the CLI reference; check `/safe-mode/take ?` on the router before scripting them.

## Logging and Remote Syslog

```routeros
/system/logging/action/add name=syslog target=remote remote=192.168.88.5 \
  remote-port=514 remote-log-format=syslog
/system/logging/add topics=info action=syslog
/system/logging/add topics=ntp,debug,!packet action=syslog
```

- Rules: `topics=` (prefix `!` to exclude), `action=`, `prefix=`, `regex=` (7.17+).
- Targets: `memory`, `disk`, `echo`, `remote`, `email`, `script` (7.22+), `cmr`.
- **Over TCP/TLS use `remote-log-format=cef`:** the `default` and `syslog` formats have no message delimiter on a stream.

All remote options and memory buffers: [logging.md](./references/logging.md).

## NTP

v7 has a built-in NTP client and server (RFC 5905); the v6 SNTP client and separate `ntp` package are gone.

```routeros
/system/ntp/client/set enabled=yes servers=pool.ntp.org
/system/ntp/client/print
/system/ntp/monitor-peers
```

- `servers=` accepts FQDNs, IPv4, IPv6, `<addr>@<vrf>` and `<addr>%<interface>`.
- `use-peer-ntp=yes` on `/ip/dhcp-client` adds dynamic servers.
- `status=synchronized` in `print` means the clock is set.
- Server: `/system/ntp/server`.

## Certificates and Let's Encrypt (ACME)

```routeros
/certificate/add-acme domain-names=[/ip/cloud/get dns-name]
```

- `/certificate/add-acme` replaces the removed `enable-ssl-certificate`. Let's Encrypt is the default `directory-url` (7.23+). Renews automatically at 80 % of validity; force with `/certificate/acme-renew` (7.24+).
- **IP Cloud name** (`<id>.sn.mynetname.net`, 7.16+): DNS-01 through MikroTik Cloud, **no port 80 needed**. Wildcards work (fixed 7.25). Requires DDNS on (see below).
- **Own domain:** HTTP-01, needs `www` enabled in `/ip/service` and TCP 80 reachable from the internet.
- **Built-in trust store (7.19+):** `/certificate/settings builtin-trust-store=default` (7.22.1+) lets `fetch`, `mqtt`, `email`, `netwatch`, `container`, `dns`, `www` and other clients verify public servers without importing CAs.

CA creation, import/export, trust store details: [certificates.md](./references/certificates.md).

## IP Cloud DDNS (`/ip/cloud`)

```routeros
/ip/cloud/set ddns-enabled=yes
/ip/cloud/force-update
/ip/cloud/print
```

- Talks to `cloud2.mikrotik.com` over **UDP 15252**; publishes A and AAAA for `<serial>.sn.mynetname.net`.
- `ddns-enabled=auto` (default since 7.17, previously `no`) turns DDNS on only while Back To Home is enabled. Use `yes` for always-on.
- `ddns-update-interval` forces periodic checks (needed when the WAN IP is behind upstream NAT).
- `update-time=yes` (default) sets the clock from Cloud at boot **only if** the NTP client is disabled. Pair with `/system/clock/set time-zone-autodetect=yes`.
- Not available on x86; CHR needs a paid perpetual license.
- Back To Home VPN: see `routeros-vpn`.

## Scheduler

```routeros
/system/scheduler/add name=nightly start-time=03:00:00 interval=0s \
  days=mon,tue,wed,thu,fri on-event="/system/script/run backup"
```

- `start-time=startup` runs shortly after boot **only when `interval=0s`**. With a non-zero interval the first run is one interval after boot.
- Startup jobs run before interfaces and DHCP are up; poll or delay if they need the network.
- `days=` (7.24+): `never`, `always` or a list of `sun..sat`. Each matching day restarts at `start-time` and repeats every `interval` until the end of the day. **`interval` >= `1d` is not supported with `days=`**; use `interval=0s` for once per matching day.
- The scheduler is disabled in `device-mode` `home` unless `scheduler=yes`.
- An entry runs with its own `policy=`; calling a script that needs more policies fails unless you use `/system/script/run <name> use-script-permissions`. More in `routeros-scripting`.

## Gotchas

- **Package changes need `apply-changes`.** `/system/reboot` drops them.
- **Firmware lags OS.** Check `/system/routerboard` after every upgrade.
- **`/export` is not a full backup:** no users, no certificates, secrets only with `show-sensitive`.
- **Binary backups carry MACs:** do not restore them onto other hardware.
- **Safe Mode past 100 actions** silently commits.
- **`run-after-reset` scripts** are killed after 2 minutes.
- **Syslog over TCP/TLS** needs `cef` format.
- **Scheduler `startup` with an interval** does not run at boot.

## Deprecated / Removed

| Item | Replacement | Version |
|---|---|---|
| `/system/upgrade` | `/system/package/local-update` | 7.17 |
| `/certificate/enable-ssl-certificate` | `/certificate/add-acme` (old command returns an error) | v7 |
| `/certificate/settings builtin-trust-anchors` | `builtin-trust-store` | 7.21 |
| v6 SNTP client / `ntp` package | Built-in `/system/ntp` | v7 |
| `lora` package | Merged into `iot`; package removed in 7.20 | 7.11 / 7.20 |
| MetaROUTER | Removed | 7.15 |

## References

- [backup-export.md](./references/backup-export.md): `/export` flags, `/import` behaviour, `*.auto.rsc`, cloud backup
- [logging.md](./references/logging.md): logging actions, remote syslog options, memory buffers, script target
- [certificates.md](./references/certificates.md): ACME, trust store, CA templates, import/export

Official docs:
- Upgrade: <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/upgrade>
- Packages: <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/packages>
- Configuration management (export, import, Safe Mode): <https://manual.mikrotik.com/docs/getting-started/configuration-management/>
- Backup: <https://manual.mikrotik.com/docs/getting-started/configuration-management/backup>
- Reset configuration: <https://manual.mikrotik.com/docs/cli-reference/system/reset-configuration>
- Safe mode: <https://manual.mikrotik.com/docs/cli-reference/safe-mode>
- NTP: <https://manual.mikrotik.com/docs/system-information-and-utilities/ntp>
- Certificates: <https://manual.mikrotik.com/docs/authentication-authorization-accounting/certificates>
- Cloud: <https://manual.mikrotik.com/docs/network-management/cloud>
- Scheduler: <https://manual.mikrotik.com/docs/system-information-and-utilities/scheduler>
