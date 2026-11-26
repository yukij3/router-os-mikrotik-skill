# Logging (`/system/logging`)

Referenced from `routeros-maintenance` SKILL.md.

## Rules (`/system/logging`)

| Property | Notes |
|---|---|
| `topics` | Comma list; prefix `!` to exclude, e.g. `topics=ntp,debug,!packet` |
| `action` | Name of a logging action |
| `prefix` | Text prepended to messages |
| `regex` | 7.17+: only log messages matching the regex |

## Actions (`/system/logging/action`)

`target=memory|disk|echo|remote|email|script|cmr`

### Memory

- Several `target=memory` actions can exist; each keeps its own buffer named after the action.
- View: `/log/print where buffer=<name>`.
- Clear (7.20+): `/system/logging/action/clear action=<name>`.

### Script (7.22+)

`target=script` runs a `/system/script` for every message, with `$topics` and `$message` available to the script.

### Remote syslog

```routeros
/system/logging/action/add name=syslog target=remote remote=192.168.88.5 \
  remote-port=514 remote-log-format=syslog syslog-time-format=iso8601
/system/logging/add topics=info action=syslog
```

| Property | Notes |
|---|---|
| `remote` | IPv4, IPv6 or hostname (7.17+) |
| `remote-port` | Default `514` |
| `vrf` | 7.19.6+ |
| `remote-protocol` | `udp`, `tcp` (7.18+), `tls` (7.23+, with `check-certificate=`) |
| `remote-log-format` | `default`, `syslog`, `cef` (7.18+) |
| `syslog-time-format` | `bsd-syslog`, `iso8601` (millisecond precision 7.18+) |
| `add-topics-string` | `yes`/`no` (7.22+) |
| `syslog-facility`, `syslog-severity` | |

**TCP/TLS framing:** `default` and `syslog` formats have no message delimiter, so a stream receiver cannot split messages. Use `remote-log-format=cef` with `remote-protocol=tcp` or `tls`.

Docs: <https://manual.mikrotik.com/docs/cli-reference/system/logging/action/>
