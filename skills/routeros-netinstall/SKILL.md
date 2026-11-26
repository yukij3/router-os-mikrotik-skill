---
name: routeros-netinstall
description: "MikroTik Netinstall for RouterOS device flashing: netinstall-cli (Linux), the Windows GUI, and the RouterOS netinstall package (/tool/netinstall, 7.24+) that turns a router into a Netinstall/FlashFig server. Verified against RouterOS 7.25.2 stable / 7.24.7 long-term. Use when: automating netinstall, writing scripts that invoke netinstall-cli, building netinstall tooling, reinstalling or recovering devices from another MikroTik router, understanding etherboot/BOOTP/TFTP, working with RouterOS package files (.npk), using a mode script (-sm) or configure script (-s), migrating ARM to ARM64, or when the user mentions netinstall, /tool/netinstall, etherboot, FlashFig, or device flashing."
---

# RouterOS Netinstall

> Verified against RouterOS **7.25.2 stable** and **7.24.7 long-term**, checked 2026-11-26. Use a `netinstall-cli` version at least as new as the RouterOS you install; several flags need 7.20–7.24.

This skill focuses on **official Netinstall / `netinstall-cli` behavior**, plus the RouterOS-side `netinstall` package (7.24+). `tikoci/netinstall` is a useful wrapper and source of grounded examples, but it is only one way to drive the tool.

## What `netinstall-cli` Does

Netinstall reinstalls RouterOS onto a device that has booted into **etherboot** mode. The Linux tool, `netinstall-cli`, listens for BOOTP requests and then sends the RouterOS boot image and selected `.npk` packages.

Grounded behavior from MikroTik docs:

- Netinstall **re-formats the system drive**
- It **does not erase the RouterOS license key**
- It **does not reset RouterBOOT settings**
- Since 7.17 it **saves and restores the device-mode configuration** across the format (unless a 7.22+ mode script `-sm` changes it)
- It works over a direct **Layer 2** path using **BOOTP/DHCP ports** and **TFTP**
- It requires **root / sudo**

`netinstall-cli` is the Linux command-line variant. The Windows GUI exposes nearly the same core options.

## Command Syntax

```text
netinstall-cli [-r] [-e] [-b] [-m [-o]] [-f] [-c] [-v] [--reboot] [--shutdown]
               [-k <keyfile>] [-s <userscript>] [-sm <modescript>]
               [--mac <mac address>] {-i <interface> | -a <client-ip>} [PACKAGE]+
```

## Flags

| Flag | Meaning |
|---|---|
| `-r` | Reinstall and apply the default-configuration stage |
| `-e` | Reinstall with empty configuration |
| `-b` | Discard the currently installed branding package |
| `-m` | Enable repeated installs in one run (multi-device) |
| `-o` | With `-m`, only reinstall a given MAC once per run (7.17); without `-m` it has no effect because netinstall-cli exits after one install anyway |
| `-f` | Ignore storage-size checks |
| `-v` | Verbose output |
| `-c` | Allow multiple netinstall instances on the same host |
| `-k <keyfile>` | Install a license key (`.KEY`) |
| `-s <userscript>` | Install a persistent **configure script** that replaces the RouterOS-supplied default configuration script |
| `-sm <modescript>` | Install a one-time **mode script** for the first boot after install |
| `--mac <mac>` | Only respond to this MAC address (7.20) |
| `--reboot` | Reboot the device after installation (7.24) |
| `--shutdown` | Shut the device down after installation (7.24) |
| `-i <interface>` | Bind to a specific interface |
| `-a <client-ip>` | Assign a specific client IP; if `-i` is used, server IP is auto-detected |

## Hard Rules

1. **The system package must be listed first.** Put `routeros-...npk` first in the package list.
2. **Root privileges are required.** Netinstall uses privileged BOOTP/TFTP ports.
3. **Multi-arch package sets are allowed.** Netinstall detects the device architecture and only uses matching packages.
4. **Package architecture is checked** against the device (7.16).
5. **Other Netinstall servers are detected** on startup (7.19) — stop them or use `-c` deliberately.
6. **No `-r` and no `-e` means "keep old configuration".** Netinstall downloads the current configuration database, reformats the device, and uploads that configuration back. This does **not** preserve user files or databases such as Dude or User Manager.

## netinstall-cli Version History

| Version | Change |
|---|---|
| 7.16 | Multi-device install, mixed-architecture package sets, device/package arch check |
| 7.17 | `-o`; device-mode saved and restored on format; Windows GUI "Get key" button removed |
| 7.19 | Detects other running Netinstall servers; clears old config before running the `-s` user script |
| 7.20 | `--mac`, `-m` |
| 7.22 | `-sm <modescript>` |
| 7.24 | `--reboot`, `--shutdown`, `help` parameter |
| 7.25 | `branding-and-custom-packages` parameter for custom NPK packages (exact flag syntax not yet documented — check `netinstall-cli help`) |
| 7.26beta (development) | Installs the version of the configured update channel; can force-stop in-progress installs |

## Install Workflow and Script Order

The official workflow is:

1. Put the device into **etherboot**
2. Run Netinstall with the desired packages and optional scripts
3. On the next boot, RouterOS runs the initial-configuration steps

For Linux `netinstall-cli`, the important first-boot order is:

1. **Mode script (`-sm`) runs first**
2. **Custom/default configuration runs after that**
3. If the mode script changes **device-mode**, the device **reboots immediately** after the mode script completes

That ordering matters: use `-sm` for first-boot state that must happen **before** default or custom configuration, especially **`/system/device-mode`** and **protected-routerboot**.

## Configure Script vs Mode Script

The docs use several names for the persistent `-s` script: **configure script**, **initial configuration**, and the custom default configuration script visible at:

```routeros
/system/default-configuration/custom-script/print
```

These two script types are different:

| Feature | Configure script (`-s`) | Mode script (`-sm`) |
|---|---|---|
| Purpose | Replace RouterOS-supplied default config script | One-time first-boot actions before config scripts |
| When it runs | As the device's default-configuration stage | On first boot after install, before custom/default config |
| Persistence | Stored on device | Auto-removed after execution |
| Survives upgrades | Yes | No |
| Later `/system reset-configuration` | Runs again after reset | Does not persist for later resets |
| Version requirement | Available in RouterOS 7.x docs | Requires **RouterOS 7.22+** and **netinstall-cli 7.22+** |
| Timeout | 120 seconds | 120 seconds |
| Reboot | — | Immediate reboot if it changed device-mode |
| File format | Regular RouterOS import file (`.rsc`) | Regular RouterOS import file (`.rsc`) |

Additional grounded details:

- Configure scripts can read `$defconfPassword` and `$defconfWifiPassword` starting with **RouterOS 7.10beta8**
- MikroTik docs explicitly suggest introducing a **delay** before configure-script execution
- If a router was netinstalled with a configure script, later `/system reset-configuration` runs that same script again until the device is re-netinstalled without it

## Package URLs and Naming

Use the normal RouterOS download tree:

```text
https://download.mikrotik.com/routeros/{version}/routeros-{version}-{arch}.npk
https://download.mikrotik.com/routeros/{version}/all_packages-{arch}-{version}.zip
https://download.mikrotik.com/routeros/{version}/netinstall-{version}.tar.gz
```

Use `download.mikrotik.com` first for all release channels. Treat `cdn.mikrotik.com` as a fallback mirror/cache, not the primary version rule.

### Architecture suffixes

| Architecture | Package form |
|---|---|
| `arm` | `routeros-7.25.2-arm.npk` |
| `arm64` | `routeros-7.25.2-arm64.npk` |
| `mipsbe` | `routeros-7.25.2-mipsbe.npk` |
| `mmips` | `routeros-7.25.2-mmips.npk` |
| `smips` | `routeros-7.25.2-smips.npk` |
| `ppc` | `routeros-7.25.2-ppc.npk` |
| `tile` | `routeros-7.25.2-tile.npk` |
| `x86` | `routeros-7.25.2.npk` |

**x86 is the naming exception:** the package filename omits the architecture suffix, but the all-packages ZIP still uses `x86`, for example `all_packages-x86-7.25.2.zip`.

## Download and Run

Official Linux quick-start pattern:

```sh
wget https://download.mikrotik.com/routeros/7.25.2/netinstall-7.25.2.tar.gz
tar -xzf netinstall-7.25.2.tar.gz

sudo ./netinstall-cli -r -i eth0 \
  routeros-7.25.2-arm64.npk \
  container-7.25.2-arm64.npk
```

Static IP on the netinstall host is strongly recommended, for example:

```sh
sudo ifconfig eth0 192.168.88.2/24
```

## Common Scripted Cases

### Empty config

```sh
sudo ./netinstall-cli -e -b -i eth0 \
  routeros-7.25.2-arm64.npk
```

### Keep old configuration

```sh
sudo ./netinstall-cli -i eth0 \
  routeros-7.25.2-arm64.npk
```

This keeps the configuration database only; it does **not** preserve files stored on the device.

### First-boot mode script

```routeros
/system/device-mode update mode=advanced container=yes
```

```sh
sudo ./netinstall-cli -r -sm modescript.rsc -i eth0 \
  routeros-7.25.2-arm64.npk \
  container-7.25.2-arm64.npk
```

This is the main documented use for `-sm`: set device mode during the first boot without requiring a later manual confirmation flow.

## Etherboot Notes

Devices must be in **etherboot** mode before Netinstall can see them. Common entry methods:

- reset button
- serial console (`Ctrl+E`)
- RouterOS setting `boot-device=try-ethernet-once-then-nand`

Netinstall uses BOOTP/DHCP ports, so avoid other DHCP sources on the same segment. The docs also call out two common failure cases:

- some USB Ethernet adapters create an extra link flap and the device is missed
- DHCP snooping can block the packets unless the Netinstall-facing port is trusted
- other DHCP/BOOTP or Netinstall servers on the segment (netinstall-cli warns since 7.19)

## RouterOS `netinstall` Package (`/tool/netinstall`, 7.24+)

Since 7.24 (first in 7.24beta1) a RouterOS device can itself act as a Netinstall and FlashFig server over L2 — no PC needed. Install the extra package `netinstall-<ver>-<arch>.npk` (available for all architectures **except `smips`**); the menu is `/tool/netinstall`. It also allows remote recovery of devices that boot via `preboot-etherboot-server`.

Key `/tool/netinstall/add` parameters:

| Parameter | Default | Notes |
|---|---|---|
| `interface` | | Interface (or bridge) facing the target devices |
| `ip-range` | | Addresses handed to booting devices |
| `mac-address` | | Restrict to one device |
| `version` | | RouterOS version to install |
| `extra-packages` | | Additional packages |
| `etherboot-image`, `etherboot-image-arch` | | Boot image selection |
| `allow-etherboot` | `yes` | Serve Netinstall to etherbooting devices |
| `allow-flashfig` | `yes` | Serve FlashFig |
| `apply-default-configuration` | `no` | Like `netinstall-cli -r` |
| `keep-old-configuration` | `yes` | Like `netinstall-cli` without `-r`/`-e` |
| `auto-reboot` | `reboot` | `none`, `reboot`, `shutdown` |
| `install-once` | `yes` | Install each device only once |
| `remove-branding` | `no` | Like `netinstall-cli -b` |
| `script-file` | | Configure script (like `-s`) |
| `mode-file` | | Mode script (like `-sm`; target must be 7.22+) |
| `wait` | `no` | |

Note the defaults differ from `netinstall-cli`: the package keeps the old configuration and installs each device once unless told otherwise.

Downloaded packages are cached in `NetinstallCache/` — in RAM on devices that have a `flash/` directory, otherwise on persistent storage:

```routeros
/tool/netinstall/settings/set cache-directory=disk1/NetinstallCache
```

Gotchas:
- If targets connect through a bridge or PoE switch, set the bridge `protocol-mode=none` or the port `edge=yes` so STP does not delay the link.
- Disable or filter `preboot-etherboot-server` rules after use, otherwise devices reinstall on every reboot.

## ARM / ARM64 Notes

- **Boot partition:** since 7.17.1/7.18, ARM devices automatically enlarge the boot partition on upgrade or Netinstall.
- **L009 ARM → ARM64 migration (7.22):** run `/system/routerboard/upgrade`, then `/system/routerboard/settings/set preferred-architecture=arm64 boot-device=try-ethernet-once-then-nand`, Netinstall the ARM64 image, and reboot. **Do not** use the backup RouterBOOT (reset-button boot) for this.
- **Alpine CPUs** (`AL73400`, `AL52400`, `AL32400`): multi-partition Netinstall fixed in RouterBOOT 7.23.

## Non-x86 Hosts

`netinstall-cli` is a Linux **i386 ELF** binary.

| Host | Practical approach |
|---|---|
| x86_64 Linux | Run it directly |
| ARM/ARM64 Linux | Use QEMU user-mode (`qemu-i386-static` or `qemu-i386`) |
| macOS | Run Linux in a VM with bridged networking |

This is where `tikoci/netinstall` is useful as a reference wrapper: it automates package download, QEMU-on-ARM, and macOS VM execution, but the underlying Netinstall behavior is still the same `netinstall-cli` flow documented above.

## Related References

- Official docs: <https://help.mikrotik.com/docs/spaces/ROS/pages/24805390/Netinstall>
- netinstall-cli (Linux): <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/netinstall/netinstall-linux>
- RouterOS netinstall package: <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/netinstall/netinstall-package>
- Reset behavior for persistent configure scripts: <https://help.mikrotik.com/docs/spaces/ROS/pages/328155/Configuration%2BManagement#ConfigurationManagement-ConfigurationReset>
- Version/channel and URL patterns: `routeros-fundamentals/references/version-parsing.md`
- Wrapper/example project: <https://github.com/tikoci/netinstall>
