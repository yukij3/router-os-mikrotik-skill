# Backup, Export and Import

Referenced from `routeros-maintenance` SKILL.md.

## Binary Backup (`/system/backup`)

```routeros
/system/backup/save name=mybackup password=<secret>
/system/backup/load name=mybackup password=<secret>
```

- With a password: encrypted, default `aes-sha256`. Without a password: saved **unencrypted** (since 6.43).
- Contains users, passwords, certificates, private keys and interface MAC addresses. Meant for the same device and RouterOS version.
- Does not contain The Dude or User-Manager databases.
- `load` reboots the router.

### Cloud backup (`/system/backup/cloud`)

- One free slot, max 15 MB, AES-encrypted, stored by MikroTik Cloud.
- Downloadable with its `secret-download-key`.
- Needs the `policy` user policy since 7.23.

## Text Export (`/export`)

```routeros
/export file=cfg
/export file=cfg show-sensitive
/export terse
/export path=/ip/firewall,!/ip/firewall/nat
```

| Flag | Effect |
|---|---|
| `compact` | Default: only non-default values (`export` == `export compact`) |
| `verbose` | Include default values |
| `terse` | Full menu path on every line (easy to grep and diff) |
| `file=<name>` | Save as `<name>.rsc` |
| `show-sensitive` | Include passwords, WireGuard/IPsec keys and other secrets |
| `path=` | 7.23+: include/exclude menus, `!` to exclude |
| `where <cond>` | 7.13+: filter entries within a menu |

**Not exported:** user accounts and passwords (groups and SSH keys are exported, but users must exist before import), certificates (use `/certificate/export-certificate`), Dude and User-Manager databases.

## Import (`/import`)

```routeros
/import file-name=cfg.rsc verbose=yes dry-run
/import file-name=cfg.rsc
/import file-name=cfg.rsc from-line=42
```

- **Additive:** commands are applied on top of the current config.
- **Stops at the first error**, e.g. a duplicate address, or interface names that differ on the target device.
- `dry-run` (7.16+) checks syntax only and requires `verbose=yes`.
- `from-line=N` starts execution at line N (e.g. to continue after fixing an error).
- Files named `*.auto.rsc` uploaded over FTP/SFTP execute automatically; output goes to `*.auto.log`.

Migration to other hardware: export with `show-sensitive`, reset the target with `no-defaults=yes`, fix interface names in the `.rsc`, import, then recreate users and import certificates.

Docs: <https://manual.mikrotik.com/docs/getting-started/configuration-management/> ; <https://manual.mikrotik.com/docs/getting-started/configuration-management/backup> ; <https://manual.mikrotik.com/docs/cli-reference/system/backup/save>
