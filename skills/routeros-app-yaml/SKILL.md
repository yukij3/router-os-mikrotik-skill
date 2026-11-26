---
name: routeros-app-yaml
description: "RouterOS /app container applications and their YAML format (7.21+ built-in apps, 7.22+ custom YAML, additions through 7.25; verified against 7.25.2 stable / 7.24.7 long-term). Use when: writing or validating RouterOS /app YAML files, using /app/setup, /app/settings, /app/add, /app/enable, /app/cleanup, working with MikroTik container apps, placeholders like [accessIP] or [secret:name], app secrets, supported-archs, app-store-urls, debugging /app validation errors or apps stuck in 'starting' / 'wait for reverse proxy', or when the user mentions /app, tikapp, or RouterOS container YAML."
---

# RouterOS /app YAML Format

> Verified against RouterOS **7.25.2 stable** and **7.24.7 long-term**, checked 2026-11-26. Minimum versions are noted per feature.

RouterOS 7.21 introduced the `/app` menu (built-in app listing and management). Custom YAML apps (`/app/add`) appeared in **7.22** (first seen in 7.22beta5), and the format gained properties in 7.23, 7.24 and 7.25. Think of it as MikroTik's opinionated alternative to `docker-compose` — but it is NOT docker-compose, with significant differences.

## What /app Is

The `/app` subsystem lets users define one or more containers as a single "application" in YAML. RouterOS parses the YAML, creates containers, volumes, networks, and config files, then manages the lifecycle.

**Key concepts:**
- Each `/app` is defined by a YAML document with services, configs, volumes, networks and secrets
- The YAML is loaded via CLI (`/app/add yaml-url=...` or `/app/add yaml=[/file/get <file> contents]`) or REST API
- Built-in apps ship with RouterOS (visible at `GET /rest/app`)
- App stores (`app-store-urls=`) provide curated collections
- Requires the `container` package and `device-mode container=yes`

## Setup and CLI

```routeros
# Interactive wizard — fills /app/settings
/app/setup

# Or set directly
/app/settings/set disk=disk1 lan-bridge=bridge

# Add a custom app from a file
/app/add yaml=[/file/get my-app.tikapp.yaml contents]

/app/enable [find name=my-app]
/app/print
```

`/app/settings` properties: `disk`, `router-ip`, `lan-bridge`, `media-path`, `download-path` (7.21.2), `show-in-webfig` (default `yes`), `auto-update` (default `no` since 7.21.2), `registry-mirrors`, `app-store-urls` (7.22).

| Command | Notes |
|---|---|
| `/app/enable`, `/app/disable` | Start/stop an app |
| `/app/update` | Update app images |
| `/app/restart` | 7.23 |
| `/app/cleanup` | **Destructive** — deletes the app's containers, data and network; keeps the disabled entry |
| `/app/remove` | Custom apps only |
| `/app/network` | App network settings |
| `/app/add` | Custom app (7.22) |
| `/app/check-update` | 7.26beta7 (development channel only) |
| `/app/settings/remove-certificate` | 7.26beta4 (development channel only) |

Per-app options include `network-outgoing-access=yes|no` (7.23; `no` adds a firewall **mangle drop rule**, which can interfere with custom mangle chains), `network-pvid` (7.22) and `use-https`.

`/system/reset-configuration keep-apps=yes` (7.22) keeps installed apps across a reset.

### What RouterOS creates

- A VETH `veth-app-<app>` attached to the `internal` bridge (`172.18.0.1/24` + masquerade) or to `lan-bridge`
- A dst-nat rule per mapped port (port conflicts on `internal` are checked, 7.23)
- Storage under `<disk>/apps/layers` and `<disk>/apps/<app>/<service>_root`
- A swap file `<disk>/swap` (~1 GiB) created automatically (7.22)
- With `use-https=yes` (the default): a reverse proxy plus cloud app certificate
- VETH IP reservations persist across stop/start (7.24); on CHR the CHR's address is shown in `ui-url` (7.24)

XFS-formatted disks are supported (7.23).

### Startup blockers

| Status | Cause |
|---|---|
| `waiting for router ip to be configured` | No `router-ip` set and no `assumed-router-ip` available |
| `wait for reverse proxy` | `use-https=yes` but no valid cloud app certificate |
| stuck in `starting` | The port named `web` (or the first TCP port) has no listener inside the container |

### Platform notes

- EN7562CT devices (e.g. hEX Refresh) only support apps whose `supported-archs` include `arm_v5` (arm32v5 images).
- `/app` on ARM32 devices: added in 7.26beta1 (development channel only).

## Critical Differences from docker-compose

| Feature | docker-compose | RouterOS /app |
|---|---|---|
| Port format | `host:container[/protocol]` | Two styles (see below) |
| Environment | `KEY=value` or list | Same, but placeholders expand |
| Volumes | Named or bind mounts | Subset — no bind mount options |
| Networks | Full docker network model | Simplified — name + external |
| Build | Full Dockerfile support | Minimal (context + dockerfile) |
| Configs | Docker configs API | Inline `content` only |
| Deploy/resources | Yes | No — not supported |
| Top-level `version:` | Deprecated (was required) | Not used |
| File extension | `.yml` / `.yaml` | `.tikapp.yaml` (convention) |

## Top-Level Properties

| Property | Type | Description |
|---|---|---|
| `name` | string | Unique identifier for the /app |
| `descr` | string | Human-readable description shown in app UI |
| `page` | string (URI) | Project homepage URL |
| `category` | string | App store classification (see below) |
| `icon` | string (URI) | Icon URL (shown in WebFig with `show-in-webfig=yes`) |
| `default-credentials` | string or null | `username:password` shown in UI |
| `url-path` | string | URL path suffix for browser access (e.g., `/admin`) |
| `auto-update` | boolean | Pull and restart containers on every boot |
| `supported-archs` | list | `arm`, `arm64`, `arm_v5`, `x86` (7.25) |
| `compatible-boards` | list | Restrict to specific boards (7.25) |
| `services` | object | **Required.** Container service definitions (≥1) |
| `volumes` | object | Named volume declarations |
| `networks` | object | Network declarations |
| `configs` | object | Config file declarations with inline content (7.23) |
| `secrets` | object | Secret declarations (7.23); marked sensitive in export (7.24) |

`credentials` and `option` appear in third-party schemas but are not documented in the official manual — avoid relying on them.

### Category

RouterOS accepts **any string** for `category` (MikroTik's own manual example uses `network`). Categories used by the built-in catalog:

```
productivity, storage, networking, development, communication,
file-management, search, video, media, media-management,
home-automation, monitoring, database, automation, ai,
messaging, radio, security, business
```

The strict JSON schema treats this list as an enum, so a new category used by RouterOS may fail strict validation until the schema is updated.

## Service Properties

Each key under `services:` defines one container. Required property: `image`.

| Property | Type | Description |
|---|---|---|
| `image` | string | Container image (omit registry to use the configured/assumed registry) |
| `container_name` | string | Explicit name; used as base for file paths under `/container` |
| `hostname` | string | Container hostname |
| `entrypoint` | string or string[] | Override default entrypoint |
| `command` | string or string[] | Override default command |
| `working_dir` | string | Working directory (7.24) |
| `ports` | array | Port mappings (see format section) |
| `environment` | object or array or null | Environment variables (`KEY=value` list or `{KEY: value}` map) |
| `volumes` | array of strings | Volume mounts (e.g., `my-vol:/data`) |
| `configs` | array of objects | Config file placements (`{source, target, mode}`) |
| `secrets` | array of strings | Secrets to expose, mounted at `/run/secrets/<name>` (7.23) |
| `dns` | — | DNS server(s) for the container (7.23) |
| `privileged` | boolean | Privileged container (7.25). Use literal `true` — `yes`/`1` are reported to be silently ignored |
| `stop_signal` | string | Signal sent on stop (7.23) |
| `restart` | enum | `no`, `always`, `on-failure`, `unless-stopped` |
| `depends_on` | array or object | Service dependency ordering |
| `devices` | array of strings | Device mappings passed to container |
| `user` | string | User to run container as |
| `security_opt` | array of strings | Security options |
| `shm_size` | string | Shared memory size |
| `stop_grace_period` | string or int | Time before SIGKILL |
| `ulimits` | object | Resource limits (e.g., `nofile: {soft: 65536, hard: 65536}`) |
| `build` | object or string | Build configuration (context, dockerfile, args) |
| `healthcheck` | object | Health check (test, interval, timeout, retries, start_period) |
| `stdin_open` | boolean | Keep stdin open |
| `expose` | array | Internal ports (not published to host) |
| `attach` | boolean | Attach to stdio |

`working_dir` is listed in the 7.24 changelog but not yet in the manual's service table; its exact behavior is unverified.

## Port Format — Two Styles

Both styles are valid; new apps from 7.23beta2+ prefer the colon style.

### Old OCI-style (pre-7.23)

```
[ip:]host_port:container_port[/tcp|/udp][:label]
```

```yaml
ports:
  - "8080:80/tcp"
  - "8443:443/tcp:https"
  - "192.168.1.1:53:53/udp:dns"
```

### New RouterOS style (7.23+)

```
[ip:]host_port:container_port[:label][:tcp|:udp]
```

```yaml
ports:
  - "8080:80:web:tcp"
  - "8443:443:https:tcp"
  - "53:53:dns:udp"
```

### Long-form (object) syntax

```yaml
ports:
  - target: 80
    published: 8080
    protocol: tcp
    name: web
    app_protocol: http
```

Name the main HTTP port `web` — RouterOS waits for a listener on it (or on the first TCP port) before the app leaves `starting`.

### IP Addresses and Placeholders in Ports

```yaml
ports:
  - "[accessIP]:[accessPort]:80/tcp:web"      # Old style with placeholders
  - "[accessIP]:[accessPort]:80:web:tcp"       # New style with placeholders
```

## Placeholders

RouterOS expands these at deploy time, in port mappings, environment values and config content:

| Placeholder | Expands to |
|---|---|
| `[accessIP]` | IP address for accessing the app from outside |
| `[accessPort]` | Primary host port for external access |
| `[accessProto]` | Protocol used to access the app |
| `[containerIP]` | IP address assigned to the container |
| `[containerInterface]` | Container interface name (≤ 15 chars) |
| `[routerIP]` | Router's own IP address |
| `[networkRouterIP]` | Router's IP on the app network (7.25) |
| `[env:variable]` | Value of env variable `variable` |
| `[env:service:variable]` | Value of env variable `variable` from service `service` |
| `[secret:name]` | Value of secret `name` |

`[accessPort2]` is not documented in the current manual — do not use it in new apps.

## Configs and Secrets

Top-level `configs:` declares inline file content; services reference them:

```yaml
configs:
  my-config:
    content: |
      server {
        listen 80;
        server_name [accessIP];
      }

services:
  web:
    image: nginx:alpine
    configs:
      - source: my-config
        target: /etc/nginx/conf.d/default.conf
        mode: 0644
```

Services list `secrets:` by name; each is mounted at `/run/secrets/<name>` and can also be referenced as `[secret:name]`. Declare them in the top-level `secrets:` object; see the official manual for the declaration fields.

## Volumes and Networks

```yaml
volumes:
  app-data: {}      # Named volume (null or empty object)

networks:
  app-net:
    name: my-network
    external: true   # Use existing RouterOS network
```

## Store Schema (app-store-urls)

RouterOS can load app collections from URLs configured via `app-store-urls=` (7.22). The store format is a **YAML array** of /app definitions:

```yaml
- name: app-one
  services:
    web:
      image: nginx:alpine
- name: app-two
  services:
    db:
      image: postgres:16
```

Store files use the `.tikappstore.yaml` extension by convention.

## REST API for /app

```typescript
// List all /app entries (built-in + custom)
const apps = await fetch(`${base}/app`, auth).then(r => r.json());

// Each entry has: .id, name, yaml (raw YAML string), and metadata

// Add a custom /app from URL
await fetch(`${base}/app`, {
  method: "PUT",
  ...auth,
  body: JSON.stringify({ "yaml-url": "https://example.com/my-app.tikapp.yaml" }),
});
```

**Note:** The `/app` endpoint requires the **container** extra package to be installed.

## JSON Schema for Validation

Community schemas (tikoci/restraml) exist in two variants:

| Schema | Purpose | Port validation | Env var names |
|---|---|---|---|
| `*.latest.json` | CI/strict validation | Regex patterns enforced | `^[A-Z_][A-Z0-9_]*$` only |
| `*.editor.json` | Editor/SchemaStore UX | No regex (allows autocompletion) | Case-insensitive |

The strict schema has regex `pattern` on port strings which **prevents VSCode autocompletion**. The editor variant removes these patterns. Schemas may lag behind new RouterOS properties.

```json
{
  "yaml.schemas": {
    "https://tikoci.github.io/restraml/routeros-app-yaml-schema.latest.json": "*.tikapp.yaml",
    "https://tikoci.github.io/restraml/routeros-app-yaml-store-schema.latest.json": "*.tikappstore.yaml"
  }
}
```

## Version History

- **7.21**: `/app` menu with built-in apps. 7.21.2: `media-path`, `download-path`; `auto-update` off by default
- **7.22**: Custom apps (`/app/add`), `app-store-urls`, automatic swap file, `network-pvid`, `reset-configuration keep-apps=yes`
- **7.23beta2**: Colon-style port format (`:tcp`/`:udp` suffix)
- **7.23**: `network-outgoing-access`, `/app/restart`, XFS disks, port-conflict check, `dns`, `stop_signal`, `secrets`, `configs`, `required-hw-devices`
- **7.24**: `working_dir`, VETH IP reservation across stop/start, secrets sensitive in export, CHR address in `ui-url`
- **7.25**: `privileged: true`, `[networkRouterIP]`, `supported-archs`, `compatible-boards`; fixed slow-board app startup
- **7.26beta** (development): `/app` on ARM32, `/app/settings/remove-certificate`, `/app/check-update`

## Common Mistakes

- **Assuming docker-compose compatibility** — not all properties are supported, some behave differently
- **Using `version:` key** — RouterOS ignores it; not needed
- **Mixing port format styles** in a single entry — each port string must use ONE style exclusively
- **`privileged: yes`** — use boolean `true`
- **No `web` port listener** — app stays in `starting`
- **`use-https` without a cloud certificate** — app stays in `wait for reverse proxy`
- **Uppercase env var names required** in strict validation — use `*.editor.json` for mixed case
- **Forgetting the container package** — `/app` returns 404 without the `container` extra package
- **Using `deploy:` or `resources:`** — not supported by RouterOS
- **Running `/app/cleanup` to "reset" an app** — it deletes the app's data

## Additional Resources

- [Examples](./references/examples.md) — minimal, full-featured, store file, ports, placeholders
- Official manual: https://manual.mikrotik.com/docs/containers/apps/
- For raw `/container` and VETH details: see the `routeros-container` skill
- For RouterOS fundamentals, CLI syntax, REST API: see the `routeros-fundamentals` skill
- For running CHR in QEMU (needed to test /app): see the `routeros-qemu-chr` skill
- MikroTik forum reference: https://forum.mikrotik.com/t/amm0s-manual-for-custom-app-containers-7-22beta/268036/22
