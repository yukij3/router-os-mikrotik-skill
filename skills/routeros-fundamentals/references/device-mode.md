# Device-Mode

> Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.
> Docs: <https://manual.mikrotik.com/docs/system-information-and-utilities/device-mode/>,
> <https://manual.mikrotik.com/docs/cli-reference/system/device-mode/>

Device-mode gates access to potentially risky features. Changing it requires physical confirmation: a **cold power cycle** or a **reset/mode button press** within `activation-timeout`. A plain `/system/reboot` does **not** confirm — it cancels the pending change.

```routeros
# View current mode and pending changes
/system/device-mode/print

# Show only settings that differ from the mode's defaults
/system/device-mode/print config

# Change mode and enable features
/system/device-mode/update mode=advanced container=yes

# After executing: physically confirm within activation-timeout
# - Press the reset/mode button, OR
# - Power cycle the device (cold boot)

# Cancel a pending update
/system/device-mode/update
```

**`mode=` wipes per-feature overrides.** `/system/device-mode/update mode=advanced` resets every flag you set earlier (e.g., `container=yes`) unless it is passed again in the same command.

**A second update cancels the first.** Running `update` with parameters while another update is pending cancels the pending one without applying the new one.

**Provisioning (7.22+):** Netinstall/FlashFig mode scripts (`-sm`) can set device-mode during provisioning, removing the manual power-cycle step. See the `routeros-netinstall` skill.

**7.25:** container swap requires `container` device-mode; WinBox gained a native Device Mode dialog.

## Modes and Factory Defaults

Four modes. The factory default depends on device type (since 7.17):

| Mode | Factory default on | Notes |
|---|---|---|
| `home` | Home routers (hAP, cAP, etc.) | Most features disabled |
| `basic` | All other device types | `home` + email, fetch, romon, scheduler, sniffer |
| `advanced` | CCR, 1100 series, CHR, devices upgraded from pre-7.17 | Previously called `enterprise` (renamed 7.17) |
| `rose` | RDS (storage) devices, 7.19+ | `advanced` + `container=yes` |

## Feature Matrix

All 19 features below are `/system/device-mode/update` properties. Every feature can be set explicitly — the matrix shows the **default** per mode.

| Property | Home | Basic | Advanced | ROSE |
|---|---|---|---|---|
| `ipsec` | yes | yes | yes | yes |
| `l2tp` | yes | yes | yes | yes |
| `pptp` | yes | yes | yes | yes |
| `smb` | yes | yes | yes | yes |
| `email` | - | yes | yes | yes |
| `fetch` | - | yes | yes | yes |
| `romon` | - | yes | yes | yes |
| `scheduler` | - | yes | yes | yes |
| `sniffer` | - | yes | yes | yes |
| `bandwidth-test` | - | - | yes | yes |
| `hotspot` | - | - | yes | yes |
| `proxy` | - | - | yes | yes |
| `socks` | - | - | yes | yes |
| `zerotier` | - | - | yes | yes |
| `container` | - | - | - | yes |
| `install-any-version` | - | - | - | - |
| `partitions` | - | - | - | - |
| `routerboard` | - | - | - | - |
| `traffic-gen` | - | - | - | - |

`ipsec`, `l2tp`, `pptp` and `smb` default to `yes` everywhere but are restricted when the device is `flagged=yes`. `traffic-gen` also gates `/tool/ping-speed` and `/tool/flood-ping` (7.17).

## Other Properties

| Property | Default | Description |
|---|---|---|
| `activation-timeout` | `5m` | Window for physical confirmation (`10s`–`1d`) |
| `flagging-enabled` | `yes` | Enable suspicious-config detection |
| `flagged` | read-only | Set when suspicious config is detected; can only be cleared with `flagged=no` |
| `attempt-count` | read-only | Update requests with parameters made so far |
| `allowed-versions` | read-only | Minimum versions the device may be downgraded to (list updated in 7.24.7 / 7.25.2) |

## Attempt Limit

Every `update` with parameters increments `attempt-count`, including refused ones. At **3**, further update requests with parameters are refused until the counter is reset by a button press or power cycle within `activation-timeout`.

> **Caveat:** A REST test on CHR 7.22.1 saw `attempt-count` reach 12+ while updates kept being accepted. It is unclear whether that was CHR-specific or caused by the reset window; treat the documented limit of 3 as authoritative and do not loop update requests. See [device-mode-rest.md](./device-mode-rest.md).
