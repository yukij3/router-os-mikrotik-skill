# RouterOS REST API Patterns

> Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.
> Docs: <https://manual.mikrotik.com/docs/developer-guides/rest-api/>,
> <https://manual.mikrotik.com/docs/developer-guides/api/>

## Verb Mapping

RouterOS REST maps HTTP verbs differently from typical REST APIs:

| HTTP Verb | RouterOS Action | CLI Equivalent | Notes |
|---|---|---|---|
| `GET` | print (list/read) | `/path/print` | Array; single object for `/*ID` and singleton menus |
| `PUT` | add (create) | `/path/add` | **Not** update. Returns `201 Created`, the full created record, and a `Location` header |
| `PATCH` | set (update) | `/path/set` | Requires `/*ID` in URL; not valid on singleton menus |
| `DELETE` | remove | `/path/remove` | Requires `/*ID` in URL |
| `POST` | command (execute) | `/path/command` | `POST /path/add` returns `200` + `{"ret":"*ID"}` |

## Authentication and Permissions

- HTTP **Basic** auth only.
- The user's group needs **both** `rest-api` **and** `api` policies. The web server logs into the internal API on the user's behalf; with only `rest-api` the request fails with `500, std failure: not allowed (9)`.
- The internal API login is cached for **10 minutes**. Changes made with `/user/set` apply on the next request; group policy changes may take up to 10 minutes (until the session renews).
- 7.21+: requests are processed only after authentication completes.

```typescript
// Basic auth — empty password
const auth = {
  headers: { Authorization: `Basic ${btoa("admin:")}` },
};

// With password
const auth = {
  headers: { Authorization: `Basic ${btoa("admin:mypassword")}` },
};
```

## Limits and Concurrency

- **60-second hard limit** per request → `400 {"detail":"Session closed","error":400,"message":"Bad Request"}`. `duration=` longer than 60 s does not extend it.
- Requests from the **same user** run serially in arrival order; different users run in parallel. A slow request stalls everything queued behind it for that user — use separate users for parallel workers.
- For longer jobs use `POST /rest/execute` **without** `as-string` (returns a job ID immediately). See [async-commands-rest.md](./async-commands-rest.md).
- `/file/read` returns at most `32768` bytes per request:

```json
POST /rest/file/read
{"file":"backup.rsc","chunk-size":"32768","offset":"0"}
```

- **No CORS.** RouterOS sends no CORS headers and answers `OPTIONS` preflight with `503`. Browser apps need a same-origin proxy.

## URL Encoding of IDs

- Do **not** percent-encode `*`: `/rest/ip/address/*1` works, `%2A1` fails with `400, no such command prefix`.
- Since 7.21 other escaped characters in IDs/names are decoded, e.g. `/rest/file/flash%2Fbackup.rsc`, `%20`.

## Common Endpoints

```typescript
// Health check (no auth needed — WebFig returns HTTP 200)
const alive = await fetch("http://HOST:PORT/").then(r => r.ok);

// System identity (singleton — returns an object)
const id = await fetch("http://HOST:PORT/rest/system/identity", auth);

// System resource (CPU, memory, uptime, version, architecture-name)
const res = await fetch("http://HOST:PORT/rest/system/resource", auth);

// All interfaces
const ifaces = await fetch("http://HOST:PORT/rest/interface", auth);

// IP addresses
const addrs = await fetch("http://HOST:PORT/rest/ip/address", auth);

// Firewall filter rules
const rules = await fetch("http://HOST:PORT/rest/ip/firewall/filter", auth);

// DNS cache
const dns = await fetch("http://HOST:PORT/rest/ip/dns/cache", auth);

// Files on the router
const files = await fetch("http://HOST:PORT/rest/file", auth);

// Installed packages
const pkgs = await fetch("http://HOST:PORT/rest/system/package", auth);
```

## Filtering, `.proplist` and `.query`

```typescript
// GET: filter by property value
await fetch(`${base}/interface?type=ether`, auth);

// Multiple filters (AND)
await fetch(`${base}/ip/address?interface=ether1&disabled=false`, auth);

// Proplist — select specific properties (reduces response size)
await fetch(`${base}/interface?.proplist=name,type,running`, auth);
```

`POST /rest/<path>/print` accepts richer options:

```json
POST /rest/interface/print
{
  ".proplist": ["name", "type"],
  ".query": ["type=ether", "type=vlan", "#|"]
}
```

- `.query` uses the binary-API query words **without** the leading `?` (stack-based; `#|`, `#&`, `#!` combine the previous terms).
- `"count-only": ""` returns only the count.
- `"oid": ""` returns SNMP OIDs.

## Singleton Menus and Bulk `set`

Menus without item lists (`/system/identity`, `/ip/dns`, `/ip/service/webserver`) reject `PATCH` with `400, missing or invalid resource identifier`. Use `POST .../set`:

```json
POST /rest/system/identity/set
{"name":"router1"}
```

`POST .../set` also accepts several IDs at once:

```json
POST /rest/ip/firewall/filter/set
{".id":"*12,*13","comment":"managed"}
```

## Reordering: `move`

Ordered menus (firewall, NAT, mangle, …) support `move`:

```json
POST /rest/ip/firewall/nat/move
{".id":"*9","destination":"*C"}
```

## POST Commands (Actions)

```typescript
// Reboot
await fetch(`${base}/system/reboot`, { method: "POST", ...auth });

// Check for updates
await fetch(`${base}/system/package/update/check-for-updates`, { method: "POST", ...auth });

// Flush DNS cache
await fetch(`${base}/ip/dns/cache/flush`, { method: "POST", ...auth });

// Run a script
await fetch(`${base}/system/script/run`, {
  method: "POST",
  ...auth,
  body: JSON.stringify({ ".id": "*1" }),
});

// Change the calling user's own password
await fetch(`${base}/password`, {
  method: "POST",
  ...auth,
  body: JSON.stringify({ "old-password": "", "new-password": "N3w", "confirm-new-password": "N3w" }),
});
```

## Error Handling

```typescript
// RouterOS returns structured errors
// { "error": 400, "message": "Bad Request", "detail": "no such command prefix" }

const response = await fetch(`${base}/ip/nonexistent`, auth);
if (!response.ok) {
  const err = await response.json();
  // err.message is the HTTP reason, err.detail carries the RouterOS error text
}
```

## Web Server Components (7.21+)

`/ip/service/webserver` enables or disables parts of the web server separately for HTTP and HTTPS. All default to `yes`; disabled parts return a `403 Forbidden` HTML page.

| HTTP | HTTPS |
|---|---|
| `index-plain`, `webfig-plain`, `graphs-plain`, `rest-plain`, `crl-plain`, `scep-plain`, `acme-plain` | `index-secure`, `webfig-secure`, `graphs-secure`, `rest-secure` |

```routeros
# REST over HTTPS only
/ip/service/webserver/set rest-plain=no
```

`www-ssl` with `certificate=none` uses anonymous DH ciphers that `curl`, browsers and Python refuse — install a certificate first.

## Binary API (`api` 8728 / `api-ssl` 8729)

- Words are length-prefixed (1–5 bytes; `0xF0` + 4 bytes is the maximum; bytes `>= 0xF8` are reserved). A sentence ends with a zero-length word.
- Login: `/login` with `=name=` and `=password=` in plain text (use `api-ssl`). The legacy MD5 challenge-response is rejected on current RouterOS even though `/login` without a password still returns `=ret=<challenge>`.
- Replies: `!re` (data), `!empty` (7.18+, empty result), `!trap` (`=category=0..7`, `=message=`), `!fatal`, `!done`.
- `.tag=<value>` multiplexes commands; `/cancel =tag=<value>` stops one. `listen` emits `!re` on changes and `=.dead=true` on delete. `getall` is an alias of `print`.
- Query words: `?name` (exists), `?-name` (absent), `?name=x`, `?<name=x`, `?>name=x`, `?#` operators (`!`, `&`, `|`, `.`, stack indices). There is **no** `~` regex in API queries.
- `.proplist` entries with an `.oid` suffix return SNMP OIDs.

> **Not yet documented:** 7.25 "expose globals in the API" and 7.26beta "direct JSON access to the API" are listed in the changelogs, but their paths and formats are not in the manual. Inspect a live router before relying on them.

## /console/inspect — Command Tree Introspection

RouterOS exposes its entire command tree via `/console/inspect`. For full details on tree traversal, node types, and schema generation, see the **`routeros-command-tree`** skill.

```typescript
// List child paths under /ip
await fetch(`${base}/console/inspect`, {
  method: "POST",
  ...auth,
  body: JSON.stringify({
    request: "child",
    path: "ip",
  }),
});
// Returns: [{type: "child", name: "address", "node-type": ...}, ...]

// Get syntax description for an argument
await fetch(`${base}/console/inspect`, {
  method: "POST",
  ...auth,
  body: JSON.stringify({
    request: "syntax",
    path: "ip,address,add,address",   // comma-separated string, NOT a JSON array
  }),
});
// Returns: [{type: "syntax", text: "IP address"}]
```

**Request flags:** `self`, `child`, `completion`, `syntax`, `highlight`, `error` — combinable with commas (`"child,completion"`).

**Path format:** Comma-separated string — `"ip,address,add"` (not `"ip.address.add"`, `"/ip/address/add"`, or a JSON array).

**Node types:** `path`, `dir`, `cmd` (command), `arg` (argument). See the `routeros-command-tree` skill for the caveat on `path` vs `dir`.

**Dangerous paths to skip:** `where`, `do`, `else`, `rule`, `command`, `on-error` — syntax lookups for args inside these have crashed the REST server.

**CLI→REST mapping:** `get`→GET, `add`→PUT (creates!), `set`→PATCH, `remove`→DELETE, others→POST.

## Known Version Differences

- **7.18+**: `!empty` reply in the binary API (zero results)
- **7.21+**: `/ip/service/webserver` component toggles; escaped characters in REST URLs decoded (except `*`)
- **7.21+**: `/app` menu exists — requires **both** the `container` package and device-mode `container=yes`; `/app/add` with YAML from 7.22
- **7.23+**: `www` supports partial content (`HTTP 206`)
- **7.25+**: fixed removal of inactive REST sessions (bug introduced in 7.21)
- **7.20.8+**: minimum for reliable API protocol streaming (observed)
