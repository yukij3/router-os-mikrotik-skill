---
name: routeros-scripting
description: "RouterOS v7 scripting language and CLI configuration idioms for .rsc files and interactive commands, including newer commands (:onerror, :retry, :serialize, :jobname, :continue/:break/:exit, in/has operators). Use when: writing or reviewing RouterOS scripts, scheduler/netwatch/on-event snippets, idempotent CLI config, :local/:global/:foreach/:do syntax, [find] selectors, print as-value handling, error handling, JSON in scripts, script permissions and policies, or when the user mentions .rsc, RouterOS script, /system/script, /system/scheduler, scripting tips, or CLI config automation."
---

# RouterOS Scripting & CLI Config

> Verified against RouterOS **7.25.2 stable** / **7.24.7 long-term**, checked 2026-11-26.
> Features newer than 7.12 are tagged with their minimum version.

RouterOS scripts (`.rsc`, `/system/script`, scheduler `on-event`, Netwatch hooks, and
interactive CLI snippets) use RouterOS's own language. It is **not shell, Lua, Python,
or Tcl**. For the syntax baseline, see [references/syntax.md](./references/syntax.md).
This skill focuses on the traps that LLMs are likely to confidently get wrong.

## Do Not Use Interactive Row Numbers in Scripts

Interactive `print` output shows temporary row numbers (`0`, `1`, `2`...) for the
current console buffer. They are **not stable object IDs** and are not usable inside
scripts.

```routeros
# WRONG in scripts — "1" is only an interactive print-buffer row number
/ip/route/set 1 gateway=3.3.3.3

# Better — internal IDs have a * prefix, but can change if objects are removed/re-added
/ip/route/set *1 gateway=3.3.3.3

# Best — resolve the target at runtime
/ip/route/set [find dst-address="0.0.0.0/0"] gateway=3.3.3.3
```

For generated config, use stable selectors such as `comment=`, `name=`, or a unique
address/list value. Internal IDs are visible in `print as-value` as `.id=*HEX`.

## Treat `[find]` Results as ID Arrays

`[find ...]` returns zero, one, or many internal IDs. Many commands accept that
array directly (`set`, `remove`, `disable`), but code that needs exactly one object
should check the count before `get`.

```routeros
:local ids [/interface/find where name="ether1"]
:if ([:len $ids] = 1) do={
  :put [/interface/get ($ids->0) name]
} else={
  :error ("expected exactly one interface, got " . [:len $ids])
}
```

Use `[find comment="my-tool-tag"]` for idempotent cleanup, not broad selectors such
as `[find dynamic=no]` that can delete unrelated user configuration.

## Quote IP Prefixes in `find` / `where`

RouterOS does aggressive type conversion, but not always in the direction you expect.
For `/ip/address`, the `address` property is stored as a string including the prefix
length. An unquoted literal like `111.111.1.1/24` is an IP-prefix value and may match
nothing.

```routeros
# WRONG — can silently return nothing
/ip/address/print where address=111.111.1.1/24

# Correct
/ip/address/print where address="111.111.1.1/24"

# Correct when value came from a variable
:local prefix 111.111.1.1/24
/ip/address/print where address=[:tostr $prefix]
```

When a query unexpectedly returns nothing, inspect the stored type:

```routeros
:put [:typeof ([:pick [/ip/address/print as-value] 0]->"address")]
```

## `print as-value` Is Usually an Array of Maps

`print as-value where ...` returns `{{...}; {...}}`, even when only one row matches.
Pick a row before accessing a property.

```routeros
# WRONG — tries to read "gateway" from the outer array
:put ([/ip/route/print as-value where gateway="ether1"]->"gateway")

# Correct — pick the first map, then read the property
:put ([:pick [/ip/route/print as-value where gateway="ether1"] 0]->"gateway")
```

For commands without `get`, `as-value` is often the only script-friendly output:

```routeros
:put ([/tool/fetch url="https://example.com/file.txt" output=user as-value]->"data")
```

## Monitor Commands Need `once do={...}`

Commands such as `monitor-traffic` are interactive loops unless told to run once.
Use `once do={}` to capture their temporary values in a script. Hyphenated field
names use `$"name-with-hyphens"`.

```routeros
/interface/monitor-traffic ether1 once do={
  :put ("rx=" . $"rx-bits-per-second")
}
```

Use `:log`, not only `:put`, for scheduler or hook scripts where no terminal is
attached.

## Globals: Declare to Modify (and on Pre-7.22 Routers)

**Since 7.22** a script or function can **read** a global variable or **call** a global
function that was defined earlier (in the same script, or in another already-run script
of the same user) without declaring it. Declaring `:global name;` is still **required to
change it with `:set`**. On routers older than 7.22, declare every global you read or call.

```routeros
:global state "ready"
:global addOne do={ :return ($1 + 1) }

:global showState do={
  # 7.22+: reading $state and calling $addOne works without declaration
  :put ("state=" . $state . " next=" . [$addOne 5])
}

:global setState do={
  :global state          # required: we modify it
  :set state $1
}
```

For scripts that must also run on long-term routers older than 7.22, keep the explicit
`:global name` lines — they are harmless on newer versions.

Use unique variable names. Inside `print ... where`, a variable named like an item
property (`type`, `dst-address`, …) resolves to the **property**, not your variable.
If you must access a hyphenated variable, use quoted syntax (`$"dst-address"`).

## Arrays Are Not JavaScript Arrays

```routeros
# Empty array: {} is a syntax error
:local arr [:toarray ""]

# Indexing and assignment use ->
:set ($arr->"name") "router1"
:put ($arr->"name")
```

String concatenation with `.` distributes over arrays unless you convert first:

```routeros
:local arr {"a"; "b"}
:put ("value=" . $arr)          # two outputs: value=a and value=b
:put ("value=" . [:tostr $arr]) # one output: value=a;b
```

Named keys are sorted alphabetically. Unnamed elements preserve insertion order but
are moved before keyed elements.

## Files Have CLI-Specific Limits

RouterOS has no direct "create empty file" or append primitive. The documented
workaround creates a file via `print file=...`, then writes `contents=`.

```routeros
/file/print file=myFile
/file/set myFile.txt contents=""
```

To append, read the existing contents, concatenate, and write the whole file back.
Large files and binary content are a poor fit for RouterOS scripts; prefer `/tool/fetch`,
REST, SCP/SFTP, or external tooling when possible.

## Error Handling: `:onerror` and `:retry` (7.13+)

`:do { } on-error={ }` still works, but `:onerror` also gives you the error text.
The error variable **must come before `do=`**, otherwise `$e` is not visible in `do`.

```routeros
:onerror e in={
  /tool/fetch url="https://example.com/x" output=user as-value
} do={
  :log warning ("fetch failed: " . $e)
}

# Retry up to 5 times, 2 s apart
:onerror e in={
  :retry command={ /tool/fetch url="https://example.com/x" output=none } delay=2 max=5
} do={ :log error $e }
```

`:onerror` returns `true` when an error happened and `false` otherwise. Since 7.22
`/tool/fetch` passes the HTTP error code and headers to `:onerror`. 7.19 added
`on-error=` to `:for` / `:foreach`. Since 7.23 a **non-existent command parameter is
a runtime error** instead of being silently ignored — old scripts with typos may now fail.

## Newer Language Features (7.12+)

| Version | Feature |
|---|---|
| 7.12 | `:jobname` (current script name), `:tonsec`, `:convert transform=` (`lc`, `uc`, `md5`, `sha512`, `reverse`, …) |
| 7.13 | `:onerror`, `:serialize` / `:deserialize` (JSON), `:grep` |
| 7.14 | `:tolf`, `:tocrlf` |
| 7.15 | `:resolve type=any\|any6\|ipv4\|ipv6`; `/system/script/run ... use-script-permissions` |
| 7.16 | DSV in `:serialize` / `:deserialize`; `import ... dry-run` |
| 7.17 | `:range <from> <to>`; `:convert from/to=num` |
| 7.20 | `:timestamp [use-tz]` (epoch time) |
| 7.21 | `:toarray delimiter=`; `:toip` / `:toip6` extract the address from a prefix |
| 7.22 | `:continue`, `:break`, `:exit`; ID comparison operators; `:tobool` from strings; global read/call without declaration |
| 7.24 | `in` / `has` operators and comparison operators for arrays; `/system/scheduler days=` |
| 7.25 | `->` accepts an array of indices/keys; `where` no longer ignores flags set to `false` |

Single-instance guard with `:jobname`:

```routeros
:if ([/system/script/job/print count-only as-value where script=[:jobname]] > 1) do={
  :error "already running"
}
```

Details and more examples: [references/syntax.md](./references/syntax.md).

## JSON: `:serialize` Converts Numeric Strings

`:serialize to=json` turns numeric strings such as `"1234"` into JSON numbers
(`1234.000000`). Pass `options=json.no-string-conversion` to keep strings — and use
the same option on `:deserialize` so `"123"` does not become a number or IP.

```routeros
:put [:serialize to=json value={id="0042"} options=json.no-string-conversion]
```

## Line Continuation and Comments

A line ending in `\` cannot also carry a trailing `# comment`, and `# comment \` does
not continue onto the next line. Both are syntax errors. Put comments on their own lines.

## Script Policies Are Part of Correctness

`/system/script/run <name>` from the CLI or the scheduler runs with the **caller's**
policies, and fails if the script's own `policy` contains anything the caller lacks.
Since 7.15, `run <name> use-script-permissions` runs with the script's own policy set
(still bounded by the caller's rights unless the script has `dont-require-permissions=yes`).

Service hooks run with a restricted policy set. DHCP, Hotspot, PPP and Traffic-Monitor
hooks are limited to `read,write,reboot,ftp,romon,test` (since 7.12); Netwatch and PPP
hooks are documented as `read,write,test,reboot`. A hook that needs `sensitive`,
`policy` or `password` must call a stored script that is allowed to run it.

```routeros
/system/script/add name=backup policy=read,write,ftp,sensitive source={
  /system/backup/save name=daily
}
```

Scheduler entries default their `policy` to the creator's script-applicable policies;
`owner` records the creator. Since 7.19 a `:return` value from a scheduler script is
no longer treated as an error.

```routeros
# 7.24+: once per weekday (interval >= 1d is not supported with days=)
/system/scheduler/add name=nightly start-time=02:00:00 interval=0s \
  days=mon,tue,wed,thu,fri on-event="/system/script/run backup"
```

Be cautious with `dont-require-permissions=yes`: it allows less-privileged callers
to run the script with the script's own policy set. Use it only for deliberately
bounded scripts.

## Validate Command Shape Before Running

For unknown attributes, flags, and package-dependent commands:

1. Prefer the MikroTik manual CLI Reference:
   <https://manual.mikrotik.com/docs/cli-reference/>
2. For a live router or exact RouterOS version, `/console/inspect` is the ground
   truth. See the `routeros-command-tree` skill.
3. When available, use `rosetta` MCP (`routeros_explain_command`,
   `routeros_command_tree`) for read-only explanation before execution.
4. If uncertain and a local CHR is available, use `quickchr exec <name> '<command>'`
   on a disposable instance before publishing the script.

## Official Docs to Review First

- RouterOS manual home: <https://manual.mikrotik.com/> — replaces the older
  Confluence-based `help.mikrotik.com` documentation.
- Scripting language reference:
  <https://manual.mikrotik.com/docs/developer-guides/scripting/>
- Scripting tips and tricks:
  <https://manual.mikrotik.com/docs/developer-guides/scripting/scripting-tips-and-tricks/>
- Scripting examples:
  <https://manual.mikrotik.com/docs/developer-guides/scripting/scripting-examples/>
- Scheduler:
  <https://manual.mikrotik.com/docs/system-information-and-utilities/scheduler/>
- CLI Reference:
  <https://manual.mikrotik.com/docs/cli-reference/> — browsable command/property
  schema organized by command path.
- For console navigation, `print` options (`order-by`, `group-by`, `show-sensitive`)
  and the REST API, see the `routeros-fundamentals` skill.
