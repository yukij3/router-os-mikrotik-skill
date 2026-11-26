# MAC-Telnet Authentication (MD5 and MTWEI)

Detail for the `routeros-mac-telnet` skill. Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.

The server announces the mode by the **length of the PASSSALT it sends**:

| Server PASSSALT length | Mode | Meaning of bytes |
|------------------------|------|------------------|
| **16** | MD5 (legacy) | 16-byte random salt |
| **49** | MTWEI / EC-SRP | 33-byte server public key ‖ 16-byte salt |

Any other length is invalid — abort. This single check is the correct, robust
way to detect the mode (do not rely on RouterOS version strings).

## Classic MD5

Simple and library-free:

```
PASSWORD value = 0x00 ‖ MD5( 0x00 ‖ password ‖ salt )      // 17 bytes total
```

- Leading `0x00`, then the 16-byte MD5 digest of (`0x00` + password bytes +
  16-byte salt). The client sends only `BEGINAUTH` in step 3 and waits for the
  server's 16-byte salt.
- Modern RouterOS no longer accepts MD5 in practice. Verified on stock CHR 7.23:
  the device offers a **16-byte salt to a classic client** (one that sends only
  `BEGINAUTH`, with no MTWEI offer) but then **rejects the MD5 proof even for
  credentials it accepts over REST/native-API** ("login failure … via
  mac-telnet" in the device log). So a 16-byte salt does **not** mean MD5 will be
  honored — you must drive MTWEI to log in. MD5 stays useful only for genuinely
  legacy/downgraded gear.

## MTWEI (EC-SRP over Curve25519) — current default

A zero-knowledge password proof: the password never crosses the wire and the
exchange proves the client knows it without revealing it. It requires an
elliptic-curve implementation. (Note: MTWEI authenticates the login but does
**not** encrypt the subsequent terminal stream — MAC-Telnet has no transport
cipher. Treat it as management-plane traffic on a trusted L2 segment.)

Wire-level flow (what differs from MD5):

1. Client generates a keypair → **33-byte client public key**. This is **not**
   the standard 32-byte X25519 wire format: MTWEI runs EC-SRP on a custom
   Curve25519-in-Weierstrass form and serializes the public point as a 32-byte
   big-endian X coordinate plus a 1-byte Y-parity flag (literal `0`/`1`, not a
   SEC1 `0x02`/`0x03` compressed-point tag). Use the reference MTWEI math, not
   an off-the-shelf X25519 API.
2. In step 3 the client sends, alongside `BEGINAUTH`, a **PASSSALT (type 1)
   block carrying `username` + `0x00` + the 33-byte client public key**. (Yes —
   control type 1 is overloaded: server→client it is the salt; client→server in
   MTWEI it is the login+pubkey.)
3. Server replies with a 49-byte PASSSALT = **33-byte server public key ‖
   16-byte salt**.
4. Client derives the SRP identity in **two SHA-256 stages**:
   `v1 = SHA256(username ‖ ":" ‖ password)`, then
   `id = SHA256(salt ‖ v1)` (16-byte salt). It runs the EC-SRP exchange over
   both public keys using `id` to produce a **32-byte PASSWORD proof**, sent with
   USERNAME + terminal blocks as in MD5.
5. Server verifies; on success sends END_AUTH.

Username caveat: the C reference strips any `+...` console-parameter suffix from
the login name **only for the MTWEI identity hash** (`strsep(..., "+")` before
`mtwei_id()`), while still sending the original login string in the USERNAME
control block. Implement this if you support RouterOS console parameters in the
login field.

The cryptographic core (point arithmetic, Weierstrass↔Montgomery conversion,
proof derivation) is non-trivial — implement against a reference rather than
from this summary:

- C: `mtwei.c` / `mtwei.h` in `haakonnessjoen/MAC-Telnet` (OpenSSL EC;
  EC-SRP per the IEEE P1363.2 draft, derived from Margin Research's PoC).
- .NET: `EcsrpEngine.cs` / `EcsrpMath.cs` in `KCTech-Lab/KC.MacTelnet`.
- m2ir: `profiles/winbox-ipc/ec-srp5-handshake.yaml` captures the related
  WinBox EC-SRP5 point encoding and is useful corroboration for the 32-byte-X +
  parity shape. Do **not** copy WinBox's later AES-128-CBC transport encryption
  into MAC-Telnet; MAC-Telnet's terminal stream is not encrypted.

Constants worth pinning in tests: client/server public key = **33 bytes**
(`MTWEI_PUBKEY_LEN`), validator/proof = **32 bytes** (`MTWEI_VALIDATOR_LEN`).

## Auth-mode pragmatics

To log into current RouterOS you must implement **MTWEI** — an MD5-only client
gets a 16-byte salt but its proof is refused (see *Classic MD5* above). A robust
client offers MTWEI in `BEGINAUTH` by default and falls back to MD5 only when the
device returns a 16-byte salt (legacy gear). `tikoci/centrs` does this — it
implements **both** MD5 and MTWEI (`src/protocols/mtwei.ts`, a dependency-free
BigInt port of `mtwei.c`) and is validated over real L2 against stock CHR 7.23.

**`END_AUTH` does not mean the login succeeded.** A *failed* login also sends
`END_AUTH`, immediately followed by a `PLAINDATA` "Login failed, incorrect
username or password" message and `END`. Confirm success only when real terminal
output (a prompt/banner) arrives; treat `END_AUTH` → "Login failed" → `END` (or
an `END` right after `END_AUTH` with no prompt) as auth failure. Grounded on CHR
7.23 via `centrs`.
