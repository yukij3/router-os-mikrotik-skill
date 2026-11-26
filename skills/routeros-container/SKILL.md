---
name: routeros-container
description: "RouterOS /container subsystem for running OCI containers on MikroTik devices (RouterOS 7.x, verified against 7.25.2 stable / 7.24.7 long-term). Use when: enabling containers on RouterOS, installing the container package, enabling device-mode container, setting up VETH/bridge networking (static, DHCP, multiple VETHs), pulling from registries or importing/exporting image tarballs (/container/save), configuring env/mount lists, memory/swap/CPU limits, healthchecks and restart policies, using /container/shell or /container/log, managing container lifecycle via CLI or REST API, or troubleshooting container start failures. Triggers: RouterOS container, /container, VETH, device-mode container, envlists, mountlists, MikroTik Docker."
---

# RouterOS Container Subsystem

> Verified against RouterOS **7.25.2 stable** and **7.24.7 long-term**, checked 2026-11-26. Many properties below appeared in 7.20–7.25; the minimum version is noted where it matters.

## Overview

RouterOS 7.x includes a container subsystem (`/container`) that runs OCI-compatible container images directly on MikroTik hardware. It is NOT Docker — it's MikroTik's own implementation with significant differences.

**Requirements:**
- RouterOS 7.x with the `container` extra package installed
- Device-mode `container=yes` (requires physical access for initial setup, except in `rose` mode or via a netinstall mode script)
- Sufficient storage (external USB disk recommended, 100+ MB/s, 10K+ random IOPS)
- ARM, ARM64, or x86 architecture (MIPS not supported for containers)
- EN7562CT devices (e.g. hEX Refresh): only `arm32v5` (`arm_v5`) images work — standard `arm` (v7) and `arm64` images fail

## Device-Mode — Physical Access Required

Container support is gated behind device-mode, which requires physical confirmation (reset button press or power cycle) to enable:

```routeros
# Enable container mode
/system/device-mode/update mode=advanced container=yes

# After executing: physically confirm within activation-timeout (default 5m)
# - Press reset button, OR
# - Power cycle the device
```

Device-mode is a general RouterOS security feature — not container-specific. Modes are `home`, `basic` (7.17+), `advanced` (formerly `enterprise`) and `rose` (7.19+). `container` is disabled by default in `home`/`basic`/`advanced`; in `rose` mode (RDS and similar storage devices) `container=yes` is the default.

Gotchas (7.17+):
- **Attempt limit:** after 3 parameterized `/system/device-mode/update` attempts without physical confirmation (`attempt-count=3`), further updates are locked (`too many unsuccessful attempts...`) until a button press or power cycle within `activation-timeout` (`10s`..`1d`, default `5m`).
- A plain `/system/reboot`, or running `/system/device-mode/update` without parameters, **cancels** the pending update. A reboot does NOT reset `attempt-count`.
- Cloud confirmation: AWS x86 (reboot/shutdown of the instance) and Hetzner ARM64 are supported (7.17).

For the full feature matrix, modes and update properties: see the [Device-mode reference](../routeros-fundamentals/references/device-mode.md) in the `routeros-fundamentals` skill.

**Mode script bypass (7.22+):** During netinstall, a mode script (`-sm`) can set device-mode on first boot, automatically triggering a reboot. Since 7.17, netinstall also preserves the existing device-mode across a reinstall. See the `routeros-netinstall` skill.

## Installing the Container Package

```routeros
# Check if container package is already installed
/system/package/print where name=container
```

**Method 1: Upload the `.npk` file** (offline)
```sh
# Upload via SCP (or Winbox drag-and-drop, WebFig file upload, FTP/SFTP)
scp container-7.25.2-arm64.npk admin@router:/
```
```routeros
/system/reboot
```

The official procedure: upload the `.npk` to the filesystem root and reboot. Uploading via FTP/SFTP with the name `*.auto.npk` installs and reboots automatically.

> **Caveat (unresolved):** on 7.22.1 it has been reported that a plain `/system/reboot` discarded the uploaded `.npk`, and `/system/package/apply-changes` (7.18+) was required instead. Official docs describe `apply-changes` only for scheduled changes (see below). If the package is not listed after a reboot, upload it again and run `/system/package/apply-changes`.

**Method 2: Online install** (requires internet)
```routeros
/system/package/update/check-for-updates
# mark container for install, then:
/system/package/enable container
/system/package/apply-changes
```

`/system/package/apply-changes` (7.18+) applies **scheduled** package changes (`/system/package/enable`, `disable`, `uninstall`) and downloads/installs online packages marked `XA` after `check-for-updates`. `/system/package/update/install` upgrades the whole system including installed extra packages.

## Networking Setup

### VETH (Virtual Ethernet)

Containers connect to RouterOS networking via VETH interfaces:

```routeros
# Static addressing
/interface/veth/add name=veth-myapp address=172.17.0.2/24 gateway=172.17.0.1
```

VETH properties (see [properties reference](./references/container-properties.md#veth) for the full list):
- `address=` accepts multiple comma-separated IPv4/IPv6 addresses; `gateway=` and `gateway6=` (7.20+; link-local `gateway6` fixed in 7.23)
- `mac-address=` (7.20) and `container-mac-address=` (7.21; default is `mac-address` + 1)
- `dhcp=yes` (7.20/7.21, default `no`): runs a DHCPv4 client using the router-side `mac-address` and hostname `<identity>-<veth name>`; leased IP and default route are applied inside the container; read-only `dhcp-address` shows the lease. DNS still comes from container settings, not DHCP.
- A container can have **multiple VETH interfaces** (`interface=` is multi-valued, 7.20+); the in-container interface name matches the RouterOS VETH name.
- Stop the container before modifying its VETH settings.

```routeros
# DHCP-addressed VETH directly on the LAN bridge (7.21+)
/interface/veth/add name=veth2 dhcp=yes
/interface/bridge/port/add bridge=bridge interface=veth2
```

For a static DHCP lease on a `dhcp=yes` VETH, match the router-side `mac-address`, not `container-mac-address`.

### Bridge Setup

```routeros
# Create a bridge for containers
/interface/bridge/add name=containers

# Add VETH to the bridge
/interface/bridge/port/add bridge=containers interface=veth-myapp

# Assign IP to bridge (acts as gateway for containers)
/ip/address/add address=172.17.0.1/24 interface=containers
```

### NAT / Firewall

```routeros
# Masquerade container traffic for internet access
/ip/firewall/nat/add chain=srcnat action=masquerade src-address=172.17.0.0/24

# Port forwarding from host to container
/ip/firewall/nat/add chain=dstnat action=dst-nat \
  dst-port=8080 protocol=tcp to-addresses=172.17.0.2 to-ports=80

# Allow container bridge in interface list (if firewall restricts)
/interface/list/member/add list=LAN interface=containers
```

### Layer 2 Networking (Bridge Mode)

For containers that need to be on the same L2 network as physical interfaces (e.g., netinstall):

```routeros
# Add both physical port and VETH to the same bridge
/interface/bridge/port/add bridge=mybridge interface=ether5
/interface/bridge/port/add bridge=mybridge interface=veth-netinstall
```

This gives the container direct L2 access to devices on ether5.

## Environment Variables and Mounts

### Inline (7.21+, preferred)

Set `env=` and `mount=` directly on `/container/add` — keeps the container self-contained:

```routeros
/container/add remote-image=pihole/pihole:latest interface=veth1 \
  env="TZ=Europe/Riga,WEBPASSWORD=secret" \
  mount="src=disk1/pihole,dst=/etc/pihole" \
  root-dir=disk1/images/pihole logging=yes
```

Mount options: `mode=rw|ro|rw,noexec|ro,noexec` (`ro` 7.20, `noexec` 7.23), e.g. `mount="src=disk1/a,dst=/a,mode=ro"`. Single files can be mounted (7.20), and mounts can override `/sys` and `/dev` (7.23).

### Named Lists

Create env vars and mounts as separate objects, then reference the list name:

```routeros
# Env list (7.20+: list=)
/container/envs/add list=MYAPP key=TZ value="Europe/Riga"
/container/envs/add list=MYAPP key=WEBPASSWORD value="secret"

# Mount list (7.21+: list= is mandatory)
/container/mounts/add list=appdata src=disk1/appdata dst=/data

# Reference from container
/container/add file=disk1/myimage.tar interface=veth1 \
  envlists=MYAPP mountlists=appdata root-dir=disk1/myapp
```

Individual entries in `envs` and `mounts` can be disabled (7.21/7.23).

**Best practice:** Always place container volumes on external disk (`disk1/`), never on internal flash storage.

### Property Name History

| Version | `/container/envs/add` | `/container/mounts/add` | `/container/add` references |
|---|---|---|---|
| 7.4–7.19 | `name=` `key=` `value=` | `name=` `src=` `dst=` | `envlist=` (singular), `mounts=` |
| 7.20 | `list=` (renamed from `name=`) | `name=` | `envlists=` (plural), `mounts=` |
| 7.21+ | `list=` | `list=` (renamed from `name=`, mandatory) | `envlists=`, `mountlists=`, inline `env=`, `mount=` |

## Container Images

### Option A: Pull from Registry

```routeros
# Registry prefix in remote-image works since 7.18
/container/add remote-image=ghcr.io/owner/app:latest interface=veth-myapp root-dir=disk1/app

# Docker Hub short name — with registry-url empty, docker.io is assumed (7.21.2+)
/container/add remote-image=library/alpine:latest interface=veth-myapp root-dir=disk1/alpine
```

`/container/config` settings:
- `registry-url` — empty means use `assumed-registry-url` (`docker.io` since 7.21.2; `https://lscr.io` was the default in 7.18). Set it only for a private/mirror registry; HTTP redirects are followed (7.18).
- `username`, `password` (sensitive) — registry credentials
- `tmpdir` — optional since 7.18; if unset, layers unpack in the parent directory of `root-dir`
- `layer-dir` — shared layer store (7.21); see `/container/layers`. Must not sit inside any container's `root-dir`.
- `memory-high`, `memory-max`, `swap-max` — global limits (see below)

Per-container `check-certificate=yes|no` (default `yes`, 7.20) controls registry TLS verification.

### Option B: Import a Local Archive

```routeros
/container/add file=disk1/myimage.tar interface=veth-myapp root-dir=disk1/myapp
```

Accepted archives:
- `docker save` / `podman save` output, including **multi-layer** images (large layer counts fixed in 7.21)
- `.tar.gz` (7.17+), zstd-compressed layers (7.22+)
- Archives exported by `/container/save` (7.24+)

Match the image architecture to the router (see Architecture Mapping), e.g. `docker save` after `docker pull --platform linux/arm64 ...`.

### Exporting: `/container/save` (7.24+)

```routeros
/container/save file=disk1/nginx.tar
```

Syntax: `/container/save file=<path> [commit] [name=]`. Runs in the background. With `commit`, the container's `root-dir` plus its settings are written as a new layer.

## Container Lifecycle

### CLI

```routeros
# Create container (7.21+ inline syntax)
/container/add file=disk1/myimage.tar interface=veth-myapp name=myapp \
  env="MY_VAR=hello" mount="src=disk1/appdata,dst=/data" \
  root-dir=disk1/myapp logging=yes start-on-boot=yes

/container/start [find name=myapp]
/container/stop [find name=myapp]
/container/print

# Per-container log buffer (7.20+, last 100 messages per container)
/container/log/print

# System log (start/stop events only when logging=yes)
/log/print where topics~"container"

# Shell into a running container (7.20/7.21+); exit with Ctrl-] (7.25+)
/container/shell [find name=myapp]

# Remove (must be fully stopped first)
/container/remove [find name=myapp]
```

Other commands:
- `/container/repull` (7.20; since 7.22 it stops, repulls and restarts automatically)
- `/container/check-update`, `/container/update` (7.21)
- `/container/kill signal=...` (7.21)
- `/container/run` (7.21, interactive)
- `/container/shell` options: `cmd=`, `user=`, `no-sh`, `timeout=`; `TERM=xterm` is set by default since 7.24

Logging changes: start/end events are logged only when `logging=yes` (7.17); environment variables are no longer printed in the startup log (7.24).

### Healthchecks and Restart Policy (7.23+)

```routeros
/container/set [find name=myapp] \
  healthcheck-cmd="wget -q -O- http://127.0.0.1/health" \
  healthcheck-interval=30s healthcheck-retries=3 \
  restart-policy=on-failure restart-max-count=5 stop-on-unhealthy=yes
```

`restart-policy=no|on-failure|always`; print flags `C` (starting, healthcheck pending), `H` (healthy), `U` (unhealthy). Defaults and the full list are in the [properties reference](./references/container-properties.md#healthcheck-and-restart-723).

### Resource Limits

- `memory-high=` (7.20) — soft limit: throttle/reclaim
- `memory-max=` (7.23) — hard limit (>= `1M`); the container is OOM-killed and its comment set to `killed due to out of memory`. Set globally in `/container/config` and/or per container; the lower value wins.
- `swap-max=` (7.24) — global + per container. Swap requires a swap file/disk; since 7.25 enabling swap again requires `device-mode container=yes`. Max swap size is capped at 10x device RAM.
- `cpu-list=` (7.20/7.21) — CPU core affinity
- `shm-size=` (7.21.2, default `64MiB`)
- `tmpfs=size@dir` e.g. `tmpfs=64M@/tmp`. Since 7.21.2, `/tmp` and `/run` are **not** tmpfs by default.

Read-only monitoring fields: `memory-current`, `swap-current`, `cpu-usage`, `container-size`, `data-size`.

### REST API

```typescript
const base = "http://192.168.1.1/rest";
const auth = { headers: { Authorization: `Basic ${btoa("admin:")}` } };

// List containers
const containers = await fetch(`${base}/container`, auth).then(r => r.json());

// Start container by ID
await fetch(`${base}/container/start`, {
  method: "POST", ...auth,
  headers: { ...auth.headers, "Content-Type": "application/json" },
  body: JSON.stringify({ ".id": "*1" }),
});

// Check status — .running field is "true"/"false" (strings!)
const status = await fetch(`${base}/container/*1`, auth).then(r => r.json());
if (status.running === "true") { /* container is running */ }

// Stop container
await fetch(`${base}/container/stop`, {
  method: "POST", ...auth,
  headers: { ...auth.headers, "Content-Type": "application/json" },
  body: JSON.stringify({ ".id": "*1" }),
});

// Delete — must be fully stopped. Poll .running and retry.
async function deleteContainer(id) {
  for (let i = 0; i < 5; i++) {
    const c = await fetch(`${base}/container/${id}`, auth).then(r => r.json());
    if (c.running === "false") {
      await fetch(`${base}/container/${id}`, { method: "DELETE", ...auth });
      return;
    }
    await new Promise(r => setTimeout(r, 3000));
  }
  throw new Error("Container did not stop in time");
}
```

### REST API Gotchas for Containers

- **`.running` field** is the status indicator — values are strings `"true"` / `"false"`, not booleans
- **`.stopped`:** the CLI shows an explicit stopped flag (`S`) since 7.20; whether REST also returns a `stopped` property is unverified — rely on `.running`
- **Delete while stopping = HTTP 400** — poll `.running` until `"false"` before DELETE
- **`file=` for local archive**, `remote-image=` for registry pull
- **`envlists=`** (plural, 7.20+) / **`mountlists=`** (7.21+) — older versions use `envlist=` / `mounts=`

## Common Start Failures

| Symptom / cause | Since |
|---|---|
| Both router `/ip/dns` servers (default DNS) empty and no `dns=` override on the container | 7.24 |
| Any mount/volume fails to mount | 7.21.2 |
| `root-dir=/`, or two containers share the same `root-dir` | 7.20 |
| `layer-dir` located inside a container's `root-dir` | 7.21 |
| Wrong image architecture for the device (e.g. non-`arm_v5` image on EN7562CT) | — |

## Container Properties

Selected `/container/add` properties — the full list with defaults is in the [properties reference](./references/container-properties.md).

| Property | Description |
|---|---|
| `name` | Container name (for `[find name=...]`) |
| `remote-image` / `file` | Registry image / local archive |
| `interface` | VETH interface(s), multi-valued 7.20+ |
| `root-dir`, `layer-dir` | Container filesystem / layer store (7.21) |
| `env`, `mount` | Inline env vars / mounts (7.21+) |
| `envlists`, `mountlists` | Named list references (7.20+ / 7.21+) |
| `cmd`, `entrypoint`, `workdir` | Override image config |
| `hostname`, `domain-name`, `dns`, `hosts` | Name resolution (`hosts=host=ip`, 7.21) |
| `logging`, `start-on-boot` | `yes`/`no` |
| `memory-high`, `memory-max`, `swap-max`, `cpu-list`, `shm-size`, `tmpfs` | Resource limits |
| `privileged` | `yes`/`no` (7.24; host isolation improved in 7.24.1) |
| `devices` | Physical device passthrough (7.20; video capture/Bluetooth 7.21, USB audio 7.23) |
| `restart-policy`, `healthcheck-*`, `stop-on-unhealthy` | 7.23+ |
| `stop-time`, `stop-signal` | Graceful stop (`stop-time` default `10s`, 7.21) |

## Architecture Mapping

When pulling from registries or building images, map RouterOS architecture to Docker platform:

| RouterOS `architecture-name` | Docker Platform |
|---|---|
| `arm` | `linux/arm/v7` |
| `arm64` | `linux/arm64` |
| `x86` | `linux/amd64` |
| EN7562CT devices (hEX Refresh) | `arm32v5` images only (`linux/arm/v5`) |

Query the router's architecture:
```typescript
const resource = await fetch(`${base}/system/resource`, auth).then(r => r.json());
const arch = resource["architecture-name"]; // "arm64", "arm", "x86"
```

## /app System

RouterOS 7.21 introduced the `/app` menu (built-in apps); custom YAML apps (`/app/add`) arrived in 7.22, and the format has grown through 7.25. `/app` builds containers, VETHs, NAT and storage for you. See the `routeros-app-yaml` skill.

Use manual `/container` setup when you need full VETH/bridge/L2 control, such as netinstall, DHCP relay, or other raw L2 workflows. Use `/app` YAML for standard multi-container applications that fit RouterOS app networking and port-forwarding conventions.

## Additional Resources

- [Container properties reference](./references/container-properties.md) — full `/container`, `/container/config` and `/interface/veth` property lists
- Related skills: `routeros-netinstall` (netinstall, device-mode automation), `routeros-app-yaml` (/app YAML), `routeros-fundamentals` (CLI, REST, scripting)
- MCP: `rosetta` tools (`routeros_search`, `routeros_get_page`, `routeros_search_properties`) for docs and property lookups
- Docs: [Container overview](https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container), [CLI reference](https://manual.mikrotik.com/docs/cli-reference/container/), [VETH](https://manual.mikrotik.com/docs/containers/veth)
