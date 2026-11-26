# RouterOS Container Properties Reference

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term (2026-11-26). Version in parentheses is when the property appeared or changed. For an exact per-version list, inspect `/container/add` on the target router (`/console/inspect`) or use the `rosetta` MCP tools.

## `/container/add` / `/container/set`

### Image and storage

| Property | Description |
|---|---|
| `name` | Container name |
| `remote-image` | Image reference; may include a registry prefix such as `ghcr.io/...` or `lscr.io/...` (7.18) |
| `file` | Local image archive (`docker save` / `podman save`, `.tar.gz` 7.17, zstd 7.22, `/container/save` output 7.24) |
| `root-dir` | Container filesystem location. Must not be `/` and must not be shared with another container (7.20) |
| `layer-dir` | Per-container layer directory (7.21). Must not be inside any `root-dir` |
| `check-certificate` | `yes`/`no`, registry TLS verification (default `yes`, 7.20) |
| `ignore-remote-image-change` | Ignore changes to the remote image |
| `labels` | Container labels (7.26beta3, development channel only) |

### Process

| Property | Description |
|---|---|
| `cmd` | Override image CMD |
| `entrypoint` | Override image ENTRYPOINT |
| `workdir` | Override working directory |
| `shell` | Shell setting for the container (see CLI reference) |
| `stop-signal` | Signal sent on stop, `1-SIGHUP` .. `31-SIGSYS` |
| `stop-time` | Grace period before kill (default `10s`, 7.21) |
| `start-on-boot` | `yes`/`no` |
| `logging` | `yes`/`no`. Start/end events logged only when `yes` (7.17); env vars no longer printed at startup (7.24) |
| `privileged` | `yes`/`no` (7.24; host isolation improved 7.24.1) |
| `devices` | Physical device passthrough (7.20; video capture and Bluetooth 7.21, USB audio 7.23) |

### Networking and naming

| Property | Description |
|---|---|
| `interface` | VETH interface(s); multiple allowed (7.20) |
| `hostname` | Container hostname |
| `domain-name` | Container domain name |
| `dns` | DNS server(s) for the container. If empty and router `/ip/dns` has no servers, start fails (7.24) |
| `hosts` | Extra `/etc/hosts` entries, `host=ip` (7.21) |

### Environment and mounts

| Property | Description |
|---|---|
| `env` | Inline env vars, `env="FOO=1,BAR=2"` (7.21) |
| `envlists` | Named env list(s) from `/container/envs` (7.20; was `envlist=` before) |
| `mount` | Inline mount, `mount="src=disk1/a,dst=/a,mode=ro"` (7.21) |
| `mountlists` | Named mount list(s) from `/container/mounts` (7.21; was `mounts=` before) |
| `tmpfs` | `size@dir`, e.g. `64M@/tmp`. No default tmpfs on `/tmp` and `/run` since 7.21.2 |
| `shm-size` | `/dev/shm` size (default `64MiB`, 7.21.2) |

### Resource limits

| Property | Description |
|---|---|
| `memory-high` | Soft limit — throttle/reclaim (7.20) |
| `memory-max` | Hard limit, >= `1M`; OOM kill sets comment `killed due to out of memory` (7.23). Lower of global/per-container wins |
| `swap-max` | Swap limit (7.24). Swap needs `device-mode container=yes` (7.25) and is capped at 10x RAM |
| `cpu-list` | CPU core affinity (7.20/7.21) |

### Healthcheck and restart (7.23)

| Property | Default | Description |
|---|---|---|
| `healthcheck-cmd` | — | Command run inside the container |
| `healthcheck-interval` | `30s` | Time between checks |
| `healthcheck-timeout` | `30s` | Per-check timeout |
| `healthcheck-start-period` | `0s` | Grace period after start |
| `healthcheck-start-interval` | `5s` | Check interval during start period |
| `healthcheck-retries` | `3` | Failures before unhealthy |
| `stop-on-unhealthy` | `no` | Stop the container when unhealthy |
| `restart-policy` | — | `no`, `on-failure`, `always` |
| `restart-interval` | `0s` | Delay between restarts |
| `restart-max-count` | `0` | Restart limit |

Read-only: `restart-count`, `memory-current`, `swap-current`, `cpu-usage`, `container-size`, `data-size`. Print flags: `C` starting (healthcheck pending), `H` healthy, `U` unhealthy, `S` stopped (7.20).

## `/container/config`

| Property | Description |
|---|---|
| `registry-url` | Registry URL. Empty = use `assumed-registry-url` (`docker.io` since 7.21.2) |
| `username`, `password` | Registry credentials (`password` is sensitive) |
| `tmpdir` | Extraction directory; if unset, parent directory of `root-dir` (7.18) |
| `layer-dir` | Shared layer directory, see `/container/layers` (7.21) |
| `memory-high`, `memory-max`, `swap-max` | Global limits |

## `/container/envs` (7.20+)

`list=`, `key=`, `value=`. Entries can be disabled (7.21/7.23). Before 7.20 the group property was `name=`.

## `/container/mounts` (7.21+)

`list=` (mandatory), `src=`, `dst=`, `mode=rw|ro|rw,noexec|ro,noexec`. Entries can be disabled. Single-file mounts (7.20), `/sys` and `/dev` overrides (7.23). Before 7.21 the property was `name=`.

## Commands

| Command | Notes |
|---|---|
| `/container/start`, `stop`, `remove`, `print` | Basic lifecycle |
| `/container/repull` | 7.20; auto stop/repull/start since 7.22 |
| `/container/check-update`, `/container/update` | 7.21 |
| `/container/kill signal=...` | 7.21 |
| `/container/run` | 7.21, interactive |
| `/container/shell [cmd=] [user=] [no-sh] [timeout=]` | 7.20/7.21; `TERM=xterm` default 7.24; `Ctrl-]` exits 7.25 |
| `/container/log` | Last 100 messages per container (7.20) |
| `/container/save file= [commit] [name=]` | Background export (7.24) |
| `/container/layers` | Shared layer store (7.21) |

## VETH

`/interface/veth/add` properties:

| Property | Description |
|---|---|
| `name` | Also the interface name inside the container |
| `address` | One or more comma-separated IPv4/IPv6 addresses |
| `gateway`, `gateway6` | IPv4 / IPv6 gateway (link-local `gateway6` fixed 7.23) |
| `mac-address` | Router-side MAC (7.20); used by the DHCP client |
| `container-mac-address` | Container-side MAC (7.21); default `mac-address` + 1 |
| `dhcp` | `yes`/`no` (default `no`). DHCPv4 client, hostname `<identity>-<veth name>`; DNS still from container settings |
| `dhcp-address` | Read-only, leased address |

Stop the container before changing its VETH. Static DHCPv4 leases for VETHs are supported (7.21) — match `mac-address`.
