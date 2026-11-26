# RouterOS `/user` REST API Reference

Reference for `/user`, `/user/group`, and `/user/ssh-keys` REST endpoints. Response shapes from docs and quickchr `provision.ts` patterns. Curl examples assume CHR on `127.0.0.1:9100` with default `admin:` credentials.

**(from docs, not lab-verified)** unless otherwise noted. Rechecked against the official docs for RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.

Docs: <https://manual.mikrotik.com/docs/authentication-authorization-accounting/user/>, <https://manual.mikrotik.com/docs/cli-reference/user/>, <https://manual.mikrotik.com/docs/cli-reference/user/group>

## User Object Shape

`GET /rest/user` returns a JSON array. Each element:

| Field | Type | Description |
|-------|------|-------------|
| `.id` | string | RouterOS internal ID (e.g. `*1`) |
| `name` | string | Username (alphanumeric, may include `_` `.` `#` `-` `@`; `*` prohibited) |
| `group` | string | Group name the user belongs to (`full`, `read`, `write`, or custom) |
| `address` | string | Allowed login address (IP/mask or IPv6 prefix, default `""` = any) |
| `disabled` | string | `"true"` or `"false"` |
| `last-logged-in` | string | Timestamp or `""` |
| `comment` | string | User comment |

**Note:** `password` is write-only — it never appears in GET responses.

## Endpoints

### GET /rest/user — List All Users

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/user
```

```json
[
  {
    ".id": "*1",
    "name": "admin",
    "group": "full",
    "address": "0.0.0.0/0",
    "disabled": "false",
    "last-logged-in": "jan/15/2025 10:30:00",
    "comment": "system default user"
  }
]
```

### GET /rest/user/*ID — Single User

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/user/*1
```

Returns a single JSON object (not array).

### PUT /rest/user — Create User

```bash
curl -s -u admin: -X PUT http://127.0.0.1:9100/rest/user \
  --data '{"name":"quickchr","password":"s3cret","group":"full"}' \
  -H "content-type: application/json"
```

Success returns the created object with all its properties (HTTP 201):

```json
{
  ".id": "*2",
  "name": "quickchr",
  "group": "full",
  "address": "",
  "disabled": "false",
  "last-logged-in": "",
  "comment": ""
}
```

**Writable properties on create:**

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | yes | Must start/end with alphanumeric. `*` prohibited. |
| `password` | string | yes | Mandatory; pass `""` explicitly for an empty password |
| `group` | string | yes | Mandatory. `full`, `read`, `write`, or a custom group |
| `address` | string | no | Restrict login source (e.g. `192.168.0.0/24`) |
| `comment` | string | no | Free-form comment |
| `inactivity-timeout` | duration | no | 7.16+: `00:01:00`–`1d00:00:00`, default `10m` (interactive console sessions) |
| `inactivity-policy` | enum | no | 7.16+: `none` \| `logout` \| `lockscreen`, default `none` |

**Error — duplicate name:**

```json
{"detail":"failure: user with the same name already exists","error":400,"message":"Bad Request"}
```

### POST /rest/user/add — Alternative Create

Equivalent to PUT. quickchr uses this form in `provision.ts`:

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/add \
  --data '{"name":"quickchr","password":"s3cret","group":"full"}' \
  -H "content-type: application/json"
```

Returns `{"ret":"*2"}` (the `.id` of the created user) on success. This differs from PUT which returns the full object.

### PATCH /rest/user/*ID — Update User

```bash
curl -s -u admin: -X PATCH http://127.0.0.1:9100/rest/user/*1 \
  --data '{"comment":"managed by quickchr"}' \
  -H "content-type: application/json"
```

Returns the updated user object on success.

#### Disable a User

```bash
curl -s -u admin: -X PATCH http://127.0.0.1:9100/rest/user/*1 \
  --data '{"disabled":"yes"}' \
  -H "content-type: application/json"
```

**Gotcha — self-disable silently no-ops:** RouterOS silently ignores a user disabling itself via REST PATCH (returns HTTP 200 but the `disabled` field stays `"false"`). Use a *different* user with `full` group to disable admin. This is implemented in quickchr's `disableAdmin()` which passes `verifyAuth` from the newly-created user.

### DELETE /rest/user/*ID — Remove User

```bash
curl -s -u admin: -X DELETE http://127.0.0.1:9100/rest/user/*2
```

Empty response on success. Returns `{"error":404,"message":"Not Found"}` if already deleted.

**Constraint:** The last user with `full` access rights cannot be removed.

### POST /rest/user/disable — Disable by Number/Name

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/disable \
  --data '{"numbers":"*1"}' \
  -H "content-type: application/json"
```

### POST /rest/user/enable — Re-enable

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/enable \
  --data '{"numbers":"*1"}' \
  -H "content-type: application/json"
```

## Password Change

### Change Own Password — POST /rest/password

Changes the password of the **currently authenticated** user:

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/password \
  --data '{"old-password":"","new-password":"N3w","confirm-new-password":"N3w"}' \
  -H "content-type: application/json"
```

Returns `[]` on success. (from docs)

**Note:** This is `/rest/password` — NOT `/rest/user/password`. It changes the calling user's own password.

### Set Another User's Password — PATCH

To set a password on another user (requires `policy` permission in the caller's group):

```bash
curl -s -u admin: -X PATCH http://127.0.0.1:9100/rest/user/*2 \
  --data '{"password":"newpass"}' \
  -H "content-type: application/json"
```

### POST /rest/user/expire-password

Forces the user to change password on next CLI/Winbox/SSH login:

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/expire-password \
  --data '{"numbers":"*2"}' \
  -H "content-type: application/json"
```

## Admin Account Behavior

### Default State (Fresh CHR)

- Username: `admin`, password: empty string
- Group: `full`
- `expired: true` flag is set

Newer MikroTik **hardware** instead ships with a unique factory password printed on the serial-number sticker / Quick Guide / device compartment.

### The `expired` Flag — REST Is Unaffected

**Critical for automation:** The `expired: true` flag on the admin account only triggers a password-change prompt at CLI, Winbox, and SSH login (`Change your password (Ctrl-C to skip)` since 7.21). **REST API and API sockets are completely unaffected.** Authenticated requests with `admin:""` succeed on a fresh CHR regardless of the expired flag.

Do NOT add workarounds for `expired: true` on REST paths. If early REST responses return unexpected data, the root cause is a **startup timing race** — not the expired flag.

### quickchr Pattern

1. Boot CHR, wait for REST readiness (`waitForBoot`)
2. Create managed user `quickchr` with generated password via `POST /rest/user/add` (as admin)
3. Install SSH key for the managed user (see below)
4. Disable admin via `PATCH /rest/user/*ID {"disabled":"yes"}` — using the new user's credentials
5. Store credentials in secret store

## User Groups — /rest/user/group

### GET /rest/user/group — List Groups

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/user/group
```

### Default Groups (cannot be deleted)

| Name | Key Policies |
|------|-------------|
| `read` | local, telnet, ssh, reboot, read, test, winbox, password, web, sniff, sensitive, api, romon, rest-api. **No** ftp, write, policy |
| `write` | Same as `read` + write. **No** ftp, policy |
| `full` | All policies including ftp, write, policy |

**Warning:** Even the `read` group includes `sensitive`, `reboot`, `sniff`, and `api`. Do not assign it to untrusted users. Create a custom group with minimal policies instead.

### Group Policy List

There are 17 policies, comma-separated in the `policy` field:

**Login:** `local`, `telnet`, `ssh`, `ftp`, `web`, `winbox`, `password`, `api`, `rest-api`, `romon`
**Config:** `reboot`, `read`, `write`, `policy`, `test`, `sensitive`, `sniff`

**REST needs both `rest-api` AND `api`.** The web server logs into the internal API on the user's behalf; a group with only `rest-api` gets HTTP `500, std failure: not allowed (9)`. The internal login is cached for 10 minutes, so group policy changes can take up to 10 minutes to reach REST sessions (changes via `/user/set` apply on the next request).

**`add` vs `set` semantics:** `/user/group/add` revokes every policy not listed. `/user/group/set` changes only the listed policies — prefix with `!` to revoke (`policy=!ftp`).

### PUT /rest/user/group — Create Custom Group

```bash
curl -s -u admin: -X PUT http://127.0.0.1:9100/rest/user/group \
  --data '{"name":"automation","policy":"local,ssh,read,write,api,rest-api,test,password"}' \
  -H "content-type: application/json"
```

(from docs, not lab-verified)

## SSH Keys — /rest/user/ssh-keys

SSH key management for public key authentication. Critical for quickchr's `exec --via=ssh` transport.

### GET /rest/user/ssh-keys — List Public Keys

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/user/ssh-keys
```

Returns array of key objects:

```json
[
  {
    ".id": "*1",
    "user": "quickchr",
    "bits": "256",
    "key-type": "ed25519",
    "fingerprint": "SHA256:xxxx...",
    "info": "quickchr@mymachine"
  }
]
```

Read-only properties: `user`, `bits`, `key-type`, `fingerprint`, `info`.

### Adding SSH Keys — Two Methods

#### Method 1: POST /rest/user/ssh-keys/add (paste key string)

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/ssh-keys/add \
  --data '{"user":"quickchr","key":"ssh-ed25519 AAAA...base64... quickchr@mymachine"}' \
  -H "content-type: application/json"
```

**Only OpenSSH format keys accepted** via `add`. Parameters:

| Property | Required | Description |
|----------|----------|-------------|
| `user` | yes | RouterOS user to associate the key with |
| `key` | yes | Full public key string in OpenSSH format |

This is the method quickchr uses in `installSshKey()` via the serial console (`/user/ssh-keys/add`), then verifies via `GET /rest/user/ssh-keys`.

#### Method 2: POST /rest/user/ssh-keys/import (from file on router)

```bash
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/ssh-keys/import \
  --data '{"user":"quickchr","public-key-file":"id_ed25519.pub"}' \
  -H "content-type: application/json"
```

Requires the public key file to already exist on the router's filesystem (uploaded via SCP/FTP). Accepts PEM, PKCS#8, or OpenSSH formats.

| Property | Required | Description |
|----------|----------|-------------|
| `user` | yes | RouterOS user to associate the key with |
| `public-key-file` | yes | Filename in router's root directory |

The key's label field is `info` (called `key-owner` before 7.21).

### DELETE /rest/user/ssh-keys/*ID — Remove Key

```bash
curl -s -u admin: -X DELETE http://127.0.0.1:9100/rest/user/ssh-keys/*1
```

### Supported Key Types

- **RSA** — PEM, PKCS#8, or OpenSSH format
- **Ed25519** — PEM, PKCS#8, or OpenSSH format
- **Ed25519-sk** — OpenSSH format (FIDO/security key), 7.21+

7.25 warns on RSA keys with a weak public exponent (`<= 7`).

**Private keys** (`/user/ssh-keys/private/import`, used by the router's SSH client) must be PKCS#1 or PKCS#8 PEM (plain or encrypted). OpenSSH-format private keys (`BEGIN OPENSSH PRIVATE KEY`) are **not** accepted — convert them first.

### SSH server changes

- 7.17: `/ip/ssh allow-none-crypto` replaced by `ciphers`
- 7.21: `/ip/ssh always-allow-password-login` replaced by `password-authentication`
- 7.24: `mlkem768x25519-sha256` key exchange added
- 7.25: default host key type is Ed25519; the SSH client verifies host keys

### SSH Key Behavior Warnings

1. **Password auth disabled by default when key exists:** Once an SSH key is added for a user, password-based SSH login is disabled for that user. Controlled by `/ip/ssh` property `password-authentication` (default: `yes-if-no-key`).

2. **Keys are not exportable:** `/export` does not include SSH keys or user passwords. They must be re-provisioned on restore.

3. **Only `full` group can change key ownership:** Changing the `user` attribute under `/user/ssh-keys/private` requires full rights.

### quickchr SSH Key Provisioning Pattern

From `provision.ts` → `installSshKey()`:

```
1. Generate ed25519 keypair with ssh-keygen
   - Store in <machineDir>/ssh/id_ed25519 (private) and .pub (public)
2. Install public key via serial console:
   /user/ssh-keys/add user="quickchr" key="ssh-ed25519 AAAA..."
   (serial is preferred over REST — commits synchronously)
3. Verify via REST:
   GET /rest/user/ssh-keys — poll until key appears for the user
4. SSH transport now works without passwords
```

**Why serial console for install:** The REST `add` endpoint may return HTTP 200 before the key is durable in RouterOS storage. The serial console command commits synchronously, making the subsequent REST verification reliable.

## Active Users — /rest/user/active

### GET /rest/user/active — List Active Sessions

```bash
curl -s -u admin: http://127.0.0.1:9100/rest/user/active
```

All properties are read-only:

| Field | Type | Description |
|-------|------|-------------|
| `.id` | string | Session ID |
| `name` | string | Username |
| `address` | string | Client IP/IPv6/MAC |
| `group` | string | User's group |
| `via` | string | Access method: `telnet`, `ssh`, `winbox`, `api`, `rest-api`, `web`, `ftp` |
| `when` | string | Login timestamp |
| `radius` | string | `"true"` if RADIUS-authenticated |

### Kill a Session — `/user/active/remove` (7.20+)

```bash
curl -s -u admin: -X DELETE http://127.0.0.1:9100/rest/user/active/*1A

# or
curl -s -u admin: -X POST http://127.0.0.1:9100/rest/user/active/remove \
  --data '{"numbers":"*1A"}' \
  -H "content-type: application/json"
```

`/user/active/request-logout` was replaced by `/user/active/remove` in 7.20; the old endpoint does not exist on current versions. (from docs, not lab-verified)

## User Settings — /rest/user/settings

Password complexity requirements:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `minimum-password-length` | integer | `0` | Minimum character length |
| `minimum-categories` | integer (0–4) | `0` | Complexity: categories = uppercase, lowercase, digit, symbol |

Failed logins are delayed since 7.20; the delay was hardened in 7.23.6 / 7.24.3 (CVE-2026-16347).

## Gotchas

1. **POST /rest/user/add vs PUT /rest/user:** Both create users. `add` returns `{"ret":"*ID"}`. `PUT` returns the full user object. quickchr uses `add`.

2. **Self-disable no-op:** A user cannot disable itself via PATCH — HTTP 200 returned but no change applied. Always use a different user.

3. **Post-boot REST race:** The `/rest/user` endpoint is subject to the same startup race as all endpoints. Briefly after boot it may return wrong data. Use a polling loop with deadline when reading users immediately after boot (see `readUser()` + `createUser()` in provision.ts).

4. **Never percent-encode `*` in `.id`:** `PATCH /rest/user/*1` works; `%2A1` fails with `400, no such command prefix`. URL constructors leave `*` alone (it is an RFC 3986 sub-delimiter), but manual encoders may not.

5. **`numbers` parameter in action endpoints:** `disable`, `enable`, `expire-password` accept `numbers` which takes the `.id` value (e.g. `"*1"`), not the username string.

6. **All values are strings:** Consistent with all RouterOS REST responses — booleans are `"true"`/`"false"`, even disabled is a string.

> **Source:**
> - MikroTik manual: [User](https://manual.mikrotik.com/docs/authentication-authorization-accounting/user/), [REST API](https://manual.mikrotik.com/docs/developer-guides/rest-api/), [SSH](https://manual.mikrotik.com/docs/management-tools/ssh), [/user CLI reference](https://manual.mikrotik.com/docs/cli-reference/user/)
> - Code: `quickchr/src/lib/provision.ts` — createUser, disableAdmin, installSshKey patterns
> - Instruction: `provisioning.instructions.md` — admin expired caveat, SSH key provisioning
> - Instruction: `general.instructions.md` — admin expired note
