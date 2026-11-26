# RouterOS Extra Packages

> Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.
> Docs: <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/packages/>,
> <https://manual.mikrotik.com/docs/cli-reference/system/package/>

## Overview

RouterOS ships with the base `routeros` package. Additional functionality comes from **extra packages**
(`.npk` files). They are either fetched from MikroTik's download server by the router itself, or
uploaded manually.

## Package Installation

### From MikroTik's download server (7.18+)

```routeros
# 1. Query the download server; available packages appear with the "A" flag
/system/package/update/check-for-updates

# 2. Schedule the desired package
/system/package/enable container

# 3. Commit: downloads the package, applies changes and reboots
/system/package/apply-changes
```

The packages are **not** preinstalled in the image: `check-for-updates` asks the download server
which packages exist for this version/architecture, and `enable` + `apply-changes` downloads them.
The router needs HTTP/HTTPS access to MikroTik's servers (or a `/system/package/local-update`
mirror).

⚠️ **On 7.18+ a plain `/system/reboot` discards scheduled enable/disable/uninstall changes.**
Always use `/system/package/apply-changes`. Since 7.24 `apply-changes` no longer requires the
`sensitive` policy. On versions before 7.18 there is no `apply-changes`; `/system/reboot` applies
pending changes there.

See [packages-rest.md](./packages-rest.md) for REST details and response shapes.

### Manual upload (offline routers, third-party `.npk`)

```routeros
# 1. Upload the .npk (SCP/SFTP, WinBox drag-and-drop, or WebFig file upload)
# scp container-7.25.2-arm64.npk admin@router:/

# 2. Reboot — manually uploaded .npk files are installed on reboot
/system/reboot
```

Uploading a package renamed to `<pkg>.auto.npk` via FTP/SFTP triggers the reboot and install
automatically.

## Extra Packages per Architecture

23 extra packages (`x86` packages are also used on CHR):

| Package | Architectures | CLI paths / notes |
|---|---|---|
| `calea` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | `/system/calea` — lawful intercept |
| `cmr` | arm, arm64, x86 | |
| `container` | arm, arm64, x86 | `/container`, `/app` — also needs device-mode `container=yes` |
| `dude` | arm, arm64, mmips, tile, x86 | The Dude server |
| `extra-nic` | arm64 | Third-party NIC drivers for ARM64 UEFI systems |
| `gps` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | `/system/gps` |
| `hotspot` | smips | Separate package only on `smips`; built into `routeros` on other architectures |
| `iot` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | `/iot` — MQTT, BLE, LoRa |
| `iot-bt-extra` | arm, arm64 | USB Bluetooth LE 4.0+ adapters |
| `netinstall` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | 7.24+: turns the device into a Netinstall server |
| `openflow` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | 7.20+ |
| `rose-storage` | arm, arm64, tile, x86 | `/disk` — extended storage features |
| `switch-marvell` | arm64 | 7.19+, CRS8xx |
| `tr069-client` | arm, arm64, mipsbe, mmips, smips, tile, ppc, x86 | `/tr069-client` — TR-069/CWMP |
| `ups` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | `/system/ups` |
| `user-manager` | arm, arm64, mipsbe, mmips, tile, ppc, x86 | `/user-manager` — RADIUS server |
| `wifi-halow-mm` | arm | 802.11ah (HaLow) |
| `wifi-mediatek` | arm | `/interface/wifi` — MediaTek Wi-Fi driver (7.19+) |
| `wifi-qcom` | arm, arm64 | `/interface/wifi` — Qualcomm 802.11ax (Wi-Fi 6/6E), formerly `wifiwave2` |
| `wifi-qcom-ac` | arm | `/interface/wifi` — Qualcomm 802.11ac (Wi-Fi 5) boards |
| `wifi-qcom-be` | arm64 | `/interface/wifi` — Qualcomm Wi-Fi 7 (802.11be; MLO, 320 MHz) |
| `wireless` | arm, arm64, mipsbe, mmips, smips, tile, ppc, x86 | `/interface/wireless`, `/caps-man` — legacy 802.11a/b/g/n/ac and 60 GHz driver + legacy CAPsMAN (separate package since 7.13) |
| `zerotier` | arm, arm64 | `/zerotier` — **not** available on x86/CHR |

Wireless driver packages are mutually exclusive: since 7.18 RouterOS blocks installing more than
one at a time (e.g., `wireless` conflicts with `wifi-qcom` / `wifi-qcom-ac`). See the
`routeros-wifi` skill for which package fits which board.

Some packages are also gated by device-mode (`container`, `zerotier`, `hotspot`, …) — installing
the package alone is not enough. See [device-mode.md](./device-mode.md).

## Download URL

Extra packages are bundled in a single zip per architecture:

```
https://download.mikrotik.com/routeros/{version}/all_packages-{arch}-{version}.zip
```

Architectures: `x86`, `arm64`, `arm`, `mipsbe`, `mmips`, `smips`, `ppc`, `tile`

**x86 naming exception:** Individual x86 `.npk` files omit the architecture suffix entirely
(e.g., `routeros-7.25.2.npk`, `container-7.25.2.npk`). The `all_packages` zip does use `x86` in its
name (`all_packages-x86-7.25.2.zip`). See [version-parsing reference](./version-parsing.md) for
full download URL patterns.

## Impact on Command Tree

Installing extra packages **extends the command tree** — new paths, commands, and arguments
become visible via `/console/inspect`. This is why schema generation runs in two variants:
- **Base**: only built-in RouterOS commands
- **Extra**: all packages installed — captures the full command tree

The `/app` menu requires **both** the `container` package and device-mode `container=yes`.
Without them, `GET /rest/app` returns 404.

## Package Detection

```typescript
// Check installed packages via REST
const packages = await fetch(`${base}/system/package`, auth).then(r => r.json());
// Returns array: [{name: "routeros", version: "7.25.2", ...}, {name: "container", ...}]

const hasContainer = packages.some(p => p.name === "container" && p.disabled === "false");
```

REST values are strings, so compare `disabled` against `"false"` rather than using `!p.disabled`.
