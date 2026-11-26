# RouterOS `/system/package` REST API Reference

Response shapes captured on CHR 7.22.1 (x86_64); behavior rechecked against the official docs for RouterOS 7.25.2 stable / 7.24.7 long-term (2026-11-26).
Docs: <https://manual.mikrotik.com/docs/cli-reference/system/package/>, <https://manual.mikrotik.com/docs/getting-started/installation-and-upgrade/packages/>

## Package Object Shape

`GET /rest/system/package` returns a JSON array. Each element:

| Field | Type | Description |
|-------|------|-------------|
| `.id` | string | RouterOS internal ID (e.g. `*1`) |
| `name` | string | Package name (e.g. `routeros`, `container`) |
| `version` | string | Installed version, or `""` if not installed |
| `available` | string | `"true"` if not yet installed, `"false"` if installed |
| `disabled` | string | `"true"` if disabled/not-active, `"false"` if active |
| `scheduled` | string | Pending action (see below), or `""` |
| `build-time` | string | Build timestamp |
| `size` | string | Package size in bytes |

## Package States

| State | `available` | `disabled` | `version` |
|-------|-------------|------------|-----------|
| Installed + active | `"false"` | `"false"` | `"7.22.1"` |
| Installed + disabled | `"false"` | `"true"` | `"7.22.1"` |
| Available (not installed) | `"true"` | `"true"` | `""` |

## Available Packages Come From the Download Server

Extra packages are **not** built into the image. `check-for-updates` asks MikroTik's download server which packages exist for the running version and architecture; they then appear with `available="true"` (`A` flag in CLI). `enable` + `apply-changes` **downloads** the package, so the router needs HTTP/HTTPS access to MikroTik's servers (or a `/system/package/local-update` mirror).

On a CHR 7.22.1 (x86) the list after `check-for-updates` was: `calea`, `container`, `dude`, `gps`, `iot`, `openflow`, `rose-storage`, `tr069-client`, `ups`, `user-manager`, `wireless` (plus the installed `routeros`). The exact list depends on version and architecture — see [extra-packages.md](./extra-packages.md).

## Package Visibility

- **Fresh boot**: only installed packages appear (typically just `routeros`).
- **After check-for-updates**: all packages available on the download server for this version/arch appear in the list.
- **After disable + apply-changes**: disabled packages remain visible with `disabled="true"`.

## Endpoints

### GET /rest/system/package

Returns all visible packages.

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/system/package
```

### GET /rest/system/package/update

Returns update channel and installed version.

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/system/package/update
```

Before check-for-updates:

```json
{"channel":"stable","installed-version":"7.22.1"}
```

After check-for-updates adds `latest-version` and `status`:

```json
{"channel":"stable","installed-version":"7.22.1","latest-version":"7.22.1","status":"System is already up to date"}
```

### POST /rest/system/package/update/check-for-updates

**Async command** — returns a progressive-status array with `.section` indices:

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/system/package/update/check-for-updates \
  -X POST -H "Content-Type: application/json" -d '{}'
```

```json
[
  {".section":"0","channel":"stable","installed-version":"7.22.1","status":"finding out latest version..."},
  {".section":"1","channel":"stable","installed-version":"7.22.1","latest-version":"7.22.1","status":"System is already up to date"}
]
```

**Side effect**: reveals all available optional packages in subsequent `GET /rest/system/package`.

Like every REST call this is bound by the 60-second request limit.

### POST /rest/system/package/enable

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/system/package/enable \
  -X POST -H "Content-Type: application/json" -d '{"numbers":"container"}'
```

Response: `[]` (empty array = success). Sets `scheduled="scheduled for enable"` on the package.

### POST /rest/system/package/disable

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/system/package/disable \
  -X POST -H "Content-Type: application/json" -d '{"numbers":"container"}'
```

Response: `[]` (empty array = success). Sets `scheduled="scheduled for disable"` on the package.

### POST /rest/system/package/apply-changes

Downloads scheduled packages, applies all scheduled enable/disable/uninstall changes, and reboots. Since 7.24 it no longer requires the `sensitive` policy.

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/system/package/apply-changes \
  -X POST -H "Content-Type: application/json" -d '{}'
```

Response: `[]` — connection drops as router reboots.

## Scheduled Field Values

| Value | Meaning |
|-------|---------|
| `""` | No pending changes |
| `"scheduled for enable"` | Will be installed/enabled on apply-changes |
| `"scheduled for disable"` | Will be disabled on apply-changes |
| `"scheduled for uninstall"` | Will be removed on apply-changes |
| ``"Use `apply-changes` to proceed with install"`` | Install is pending until `apply-changes` |

Note: the enable value is `"scheduled for enable"`, NOT `"scheduled for install"`.

## Critical: apply-changes vs reboot

| Endpoint | Applies scheduled changes | Triggers reboot |
|----------|--------------------------|-----------------|
| `POST /rest/system/package/apply-changes` | ✅ Yes | ✅ Yes |
| `POST /rest/system/reboot` | ❌ No | ✅ Yes |

**This is the biggest gotcha.** On 7.18+ a plain `/system/reboot` discards pending enable/disable/uninstall changes. Always use `/system/package/apply-changes` to commit them. A plain reboot still installs manually uploaded `.npk` files (and `/system/package/downgrade`).

**⚠️ Version requirement:** `/system/package/apply-changes` was added in **RouterOS 7.18**. On versions <7.18, `/system/reboot` IS the correct (and only) method — and it DOES apply pending changes on those older versions. The "reboot discards changes" behavior is specific to 7.18+ where `apply-changes` exists. (Checked on a live CHR 7.10.)

## Device-Mode Dependency

The `container` package can be enabled and installed without device-mode. However, `/container` commands (and the `/app` menu) fail with `"not allowed by device-mode"` until device-mode is set:

```routeros
/system/device-mode/update container=yes
```

This requires a power-cycle confirmation (see device-mode reference).

## Recommended Pattern for quickchr

```
1. POST /rest/system/package/update/check-for-updates   → reveals available packages
2. POST /rest/system/package/enable {"numbers":"<name>"}  → schedule enable
3. POST /rest/system/package/apply-changes {}             → reboot + apply
4. waitForBoot()                                          → poll until REST ready
5. GET /rest/system/package                               → verify installed
```

SCP upload is **not needed** when the router can reach MikroTik's download server — the enable + apply-changes flow downloads the package. For offline routers, upload the `.npk` and reboot instead.
