# RouterOS Scripting Language

## Overview

RouterOS has its own scripting language (`.rsc` files) used for automation. It is NOT bash,
NOT Lua, NOT any standard language. It runs inside the RouterOS CLI environment.

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term (2026-11-26). Docs:
<https://manual.mikrotik.com/docs/developer-guides/scripting/>

## Variable Declaration

```routeros
# Local variable (scoped to current script/function)
:local myVar "hello"

# Global variable (persists across scripts until reboot)
:global myVar "hello"

# Variable reference
:put $myVar
```

## Data Types

| Type | Syntax | Example |
|---|---|---|
| String | `"text"` | `"hello world"` |
| Number | `123` | `42`, `0xFF` |
| Boolean | `true` / `false` / `yes` / `no` | `true` |
| IP Address | `1.2.3.4` | `192.168.1.1` |
| IP Prefix | `1.2.3.0/24` | `10.0.0.0/8` |
| Array | `{1; 2; 3}` or `{"a"; "b"}` | `{1; 2; "mixed"}` |
| Time | `1h2m3s` | `30s`, `5m`, `1d` |
| Nil | (no keyword) | absent value |

## String Operations

```routeros
# Concatenation
:local greeting ("Hello " . "World")

# Substring (pick)
:local sub [:pick "Hello" 0 3]     # → "Hel"

# Length
:local len [:len "Hello"]          # → 5

# Find
:local pos [:find "Hello World" "World"]  # → 6

# Convert to/from
:local num [:tonum "42"]
:local str [:tostr 42]
```

Newer conversion helpers (check exact argument syntax with `/console/inspect` or the
manual before use):
- `:toarray` accepts `delimiter=` (7.21); `:toip` / `:toip6` extract the address from a prefix (7.21)
- `:tolf` / `:tocrlf` convert line endings (7.14)
- `:convert transform=` (7.12): `lc`, `uc`, `lcfirst`, `ucfirst`, `crlf`, `lf`, `md5`,
  `sha512`, `rot 13`, `reverse`, `none`, plus Ed25519/X25519 key conversions
  (`ed25519-private-to-x25519-private`, `x25519-private-to-x25519-public`,
  `ed25519-private-to-ed25519-public`, `ed25519-public-to-x25519-public`)
- `:convert from=`/`to=` `byte-array` (7.15/7.16) and `num` (7.17)

## Control Flow

```routeros
# If/else
:if ($x > 10) do={
  :put "big"
} else={
  :put "small"
}

# For loop
:for i from=1 to=10 do={
  :put $i
}

# Foreach
:foreach item in=$myArray do={
  :put $item
}

# While
:while ($count < 10) do={
  :set count ($count + 1)
}

# 7.22+: :continue / :break inside loops, :exit ends the script
:foreach i in=[:range 1 10] do={
  :if ($i = 3) do={ :continue }
  :if ($i = 8) do={ :break }
  :put $i
}
```

`:range <from> <to>` (7.17+) builds an integer array. 7.19 added `on-error=` to
`:for` / `:foreach`.

**Critical syntax:** `do={...}` and `else={...}` use `=` and curly braces. No colon before `do`.

## Functions

```routeros
# Define a function (stored as a global variable)
:global myFunc do={
  :local arg1 $1
  :return ("Result: " . $arg1)
}

# Call it
:put [$myFunc "test"]
```

## Common Built-in Commands

```routeros
:put "text"                    # Print to console
:log info "message"            # Write to system log
:delay 5s                      # Sleep
:execute script="/path/to/script"  # Run another script
:resolve "example.com"         # DNS lookup
:ping 8.8.8.8 count=3         # Ping
:time { /ip/route/print }     # Measure execution time
:environment print             # Show all variables
:jobname                       # 7.12+: name of the running script
:timestamp                     # 7.20+: epoch time (UTC); add use-tz for router timezone
:tonsec $t                     # 7.12+: time value in nanoseconds
:grep script=$text pattern="error"   # 7.13+: also after=, before=, filename=
```

`:resolve` accepts `type=any|any6|ipv4|ipv6` since 7.15.

## Working with Router Config

```routeros
# Add entry and capture its ID
:local newId [/ip/address/add address=10.0.0.1/24 interface=ether1]

# Find entries
:local entries [/ip/address/find where interface=ether1]

# Get property value
:local addr [/ip/address/get $newId address]

# Set property
/ip/address/set $newId disabled=yes

# Remove
/ip/address/remove $newId
```

## Scheduler (Cron Equivalent)

```routeros
/system/scheduler/add name=my-task interval=1h \
  on-event="/system/script/run myScript"

# 7.24+: restrict to weekdays (also: always, never)
/system/scheduler/add name=workdays start-time=08:00:00 interval=1d \
  days=mon,tue,wed,thu,fri on-event="/system/script/run myScript"
```

Scheduler entries default `policy` to the creator's script-applicable policies; `owner`
records the creator. Since 7.19, a `:return` value is not treated as an error.
Docs: <https://manual.mikrotik.com/docs/system-information-and-utilities/scheduler/>

## File Operations

```routeros
# Read file content
:local content [/file/get myfile.txt contents]

# Files are stored in RouterOS flash — /file/print lists them
/file/print
```

## Error Handling

```routeros
:do {
  /ip/address/add address=invalid interface=ether1
} on-error={
  :log error "Failed to add address"
}

# 7.13+: :onerror captures the message. The variable MUST come before do=
:onerror e in={
  /ip/address/add address=invalid interface=ether1
} do={
  :log error ("add failed: " . $e)
}

# :retry inside :onerror — up to max attempts, delay seconds apart
:onerror e in={
  :retry command={ /tool/fetch url="https://example.com/x" output=none } delay=2 max=5
} do={ :log error $e }
```

`:onerror` returns `true` on error, `false` on success. Since 7.22, `/tool/fetch`
passes the error code and HTTP headers to `:onerror`. Since 7.23, passing a parameter
that does not exist is a runtime error (older versions silently ignored it).

## Comments

```routeros
# This is a comment (single-line only)
:put "hello"  # Inline comment after command
```

No multi-line comment syntax exists. Each line needs its own `#`.

A line ending with `\` (continuation) cannot have a trailing `# comment`, and a
`# comment \` line does not continue onto the next line — both are syntax errors.

## `:execute` vs `:do`

```routeros
# :do — runs inline, blocks until complete
:do { /ip/address/print } on-error={ :put "failed" }

# :execute — runs in BACKGROUND, returns immediately
# Result can be captured to file or as-string
:local jobId [:execute script="/interface/print"]

# :execute with as-string — BLOCKS (not background)
:local result [:execute script=":put hello" as-string]

# :execute with file — runs in background, writes output to file
:execute script="/export" file="backup"
```

**Key difference:** `:execute` without `as-string` runs asynchronously — the script continues immediately. With `as-string`, it blocks and returns the output. Executed scripts are limited to 64KB.

## `:parse` — Dynamic Code

```routeros
# Parse a string into an executable function
:global myFunc [:parse ":put hello!"]
$myFunc

# Useful for building commands dynamically
:local cmd ":put (1 + 2)"
:local fn [:parse $cmd]
$fn   # → 3
```

`:parse` compiles a string into a callable function. This is the only way to create "functions" in RouterOS — the `do={}` syntax for globals is syntactic sugar for `:parse`.

## Array Operations

```routeros
# Create array
:local arr {1; 2; 3; "four"}

# Named keys
:local dict {name="router1"; ip=192.168.1.1}

# Access by index (uses -> not [])
:put ($arr->0)         # → 1

# Access by key
:put ($dict->"name")   # → "router1"

# Set element value
:set ($dict->"name") "router2"

# Array length
:put [:len $arr]       # → 4

# Append to array (no built-in append — rebuild or use set)
:set arr ($arr, 5)     # append 5

# Loop with keys and values
:foreach k,v in=$dict do={
  :put "$k=$v"
}

# Loop values only
:foreach v in=$arr do={
  :put $v
}
```

Newer array features:
- 7.24: `in` and `has` operators, and comparison operators for arrays
- 7.25: `->` accepts an array of indices or keys (select several elements at once)

> The exact operand order and semantics of `in` vs `has` are not documented in the
> sources used here; test them on your version before relying on them.

**⚠️ Array key sorting:** Elements with named keys are sorted alphabetically. Elements without keys preserve insertion order but are moved before keyed elements.

**⚠️ Key names with uppercase or special chars** must be quoted: `($arr->"myKey")`.

## `:serialize` / `:deserialize` (JSON)

RouterOS 7.x supports JSON serialization:

```routeros
# Serialize array to JSON
:local data {name="test"; value=42}
:put [:serialize to=json value=$data]
# → {"name":"test","value":42}

# Pretty print
:put [:serialize to=json value=$data options=json.pretty]

# Prevent string→number conversion (default converts "1234" to 1234.000000)
:put [:serialize to=json value=$data options=json.no-string-conversion]

# Deserialize JSON string to array
:local parsed [:deserialize from=json value="{\"name\":\"test\"}"]
:put ($parsed->"name")   # → test

# Deserialize from file
:deserialize [/file/get config.json contents] from=json

# Also supports DSV (delimiter-separated values)
:put [:serialize to=dsv delimiter=";" value=$data]
```

**DSV options:** `dsv.plain` (no header), `dsv.array` (header as keys), `dsv.wrap-strings`, `dsv.remap` (merge array of dicts).

Version notes: JSON since 7.13, DSV since 7.16, `json.no-string-conversion` 7.17,
`dsv.remap` and `file-name=` 7.18, `json.time-as-duration` in 7.26beta (development).
Use `json.no-string-conversion` on `:deserialize` too, or strings like `"123"` become
numbers/IPs.

## `/system/script` — Script Repository

```routeros
# Add stored script
/system/script/add name=my-backup source={
  /export file="daily-backup"
  :log info "Backup complete"
}

# Run stored script
/system/script/run my-backup

# List scripts
/system/script/print

# Edit script source
/system/script/set my-backup source={...new code...}

# Remove
/system/script/remove my-backup
```

**Properties:** `name`, `source`, `policy`, `comment`, `dont-require-permissions`, `owner` (read-only), `run-count` (read-only), `last-started` (read-only).

## Script Permissions (Policies)

Scripts have permission policies that control what they can access:

| Policy | Allows |
|--------|--------|
| `read` | Retrieve configuration |
| `write` | Change configuration |
| `policy` | Manage users and policies |
| `reboot` | Reboot the router |
| `password` | Change passwords |
| `ftp` | FTP access, send/retrieve files |
| `sensitive` | Change "hide sensitive" parameter |
| `sniff` | Run sniffer, torch |
| `test` | Run ping, traceroute, bandwidth-test |
| `romon` | RoMON access |

**Rules:**
- `/system/script/run <name>` (CLI or scheduler) runs with the **caller's** policies and fails if the script's `policy` contains any policy the caller lacks
- 7.15+: `run <name> use-script-permissions` runs with the script's own policy set, still bounded by the caller's rights
- `dont-require-permissions=yes` lets less-privileged callers run the script with its own policies (useful for Netwatch/scheduler hooks with limited permissions)
- Service hooks are restricted: DHCP, Hotspot, PPP and Traffic-Monitor hooks to `read,write,reboot,ftp,romon,test` (since 7.12); Netwatch and PPP hooks documented as `read,write,test,reboot`

## Critical Scripting Pitfalls

These are the highest-value traps from MikroTik's current "Scripting Tips and
Tricks" docs and local CHR checks. See the parent `routeros-scripting` skill for
the shorter operational version.

### Print Buffer IDs Are Not Script IDs

Interactive `print` assigns temporary row numbers (`0`, `1`, `2`...) to the current
console buffer. A script does not know what those numbers mean.

```routeros
# WRONG in scripts
/ip/route/set 1 gateway=3.3.3.3

# Better, but still brittle if the object is removed and re-added
/ip/route/set *1 gateway=3.3.3.3

# Best
/ip/route/set [find dst-address="0.0.0.0/0"] gateway=3.3.3.3
```

### `print as-value` Returns an Array of Maps

Most `print as-value` calls return `{{...}; {...}}`, even when only one row matches.

```routeros
# WRONG
:put ([/ip/route/print as-value where gateway="ether1"]->"gateway")

# RIGHT
:put ([:pick [/ip/route/print as-value where gateway="ether1"] 0]->"gateway")
```

### Globals: Declaration Needed to Modify

Since **7.22**, scripts and functions can read a global variable or call a global
function defined earlier (same script, or another already-run script of the same user)
without `:global name;`. The declaration is still **required to `:set`** the global.
On pre-7.22 routers, every reader must declare it.

```routeros
:global myVar "test"
:global myFunc do={
  :put ("global var=" . $myVar)   # 7.22+: read works without declaration
}
:global bump do={
  :global myVar                    # required to modify
  :set myVar ($myVar . "!")
}
```

### Variable Names That Match Item Properties

Inside `print ... where` / `find where`, a variable whose name matches an item property
(e.g. `type`, `dst-address`) resolves to the **property**, not the variable. Always use
distinct names (`$wantedType`, not `$type`).

### Empty Arrays and Array Concatenation

```routeros
:local arr [:toarray ""]       # {} literal is a syntax error
:set ($arr->"name") "router1"

:local values {"a"; "b"}
:put ("value=" . $values)          # repeats prefix per element
:put ("value=" . [:tostr $values]) # one string: value=a;b
```

### Quote IP Prefixes in `find` / `where`

`/ip/address address` is stored as a string containing the prefix length. Unquoted
IP-prefix literals can silently fail to match.

```routeros
# WRONG
/ip/address/print where address=192.168.1.1/24

# RIGHT
/ip/address/print where address="192.168.1.1/24"
```

Use `[:tostr $prefix]` when the value is held in a variable.

### Monitor Commands and Hyphenated Properties

Interactive monitor commands need `once do={}` for scripts. Use `$"..."` for
temporary fields or variables whose names contain hyphens.

```routeros
/interface/monitor-traffic ether1 once do={
  :put ("rx=" . $"rx-bits-per-second")
}
```

### File Creation and Append Are Indirect

```routeros
/file/print file=myFile
/file/set myFile.txt contents=""
```

There is no direct append command: read old contents, concatenate, and write the
whole file back.

### Script Policies Affect Execution

Scheduler/Netwatch/hook policies must cover what the script does. Backups and
sensitive data require `sensitive`; file operations commonly require `ftp`. Use
`dont-require-permissions=yes` only for tightly bounded scripts.

## Important Gotchas

- **No pipes, no redirection** — can't do `cmd | grep` or `cmd > file`
- **`$` is required** to reference variables — `:put myVar` prints literal "myVar"
- **Array indexing** uses `->` not `[]` — `($arr->0)` for first element
- **String comparison** uses `=` not `==` — `(:if ($a = "test") do={...})`
- **Command substitution** uses `[...]` not `$(...)` — `:local result [/system/identity/get name]`
- **Semicolons in arrays** — `{1; 2; 3}` not `{1, 2, 3}`
- **Script line continuation** — use `\` at end of line
- **Property names with hyphens** — use quotes in find: `[find where "mac-address"="AA:BB:CC:DD:EE:FF"]`
- **Variable names are case-sensitive** — `$myVar` ≠ `$myVAR`
- **`:set` without value undefines** — `:global myVar; :set myVar` removes it from environment
- **`:execute` script size limit** — 64KB max for executed scripts
- **Global variables survive across script runs** but NOT across reboots
- **Non-existent parameters fail (7.23+)** — typos that were silently ignored now raise errors
- **`where` with `false` flags (7.25)** — filters on flags set to `false` were ignored before 7.25; results may change after upgrading

> **Source:**
> - MikroTik manual: [Scripting](https://manual.mikrotik.com/docs/developer-guides/scripting/) — language reference
> - MikroTik manual: [Scripting Tips and Tricks](https://manual.mikrotik.com/docs/developer-guides/scripting/scripting-tips-and-tricks/)
> - MikroTik manual: [Scripting examples](https://manual.mikrotik.com/docs/developer-guides/scripting/scripting-examples/)
> - MikroTik manual: [Scheduler](https://manual.mikrotik.com/docs/system-information-and-utilities/scheduler/)
> - Reference: `routeros-fundamentals/references/rest-api-patterns.md` — REST vs scripting interface differences
> - Note: `:serialize`/`:deserialize` examples are from docs, not lab-verified
