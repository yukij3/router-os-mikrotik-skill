# DoS Protection Patterns

RouterOS-specific DoS protection matchers. Referenced from `routeros-firewall` SKILL.md. Verified against RouterOS 7.25.2 stable / 7.24.7 long-term.

## Official DDoS Pattern — Detect in Filter, Drop in Raw

The pattern from MikroTik's DDoS Protection guide detects floods with `dst-limit` and drops attacker/target pairs in `/ip/firewall/raw` `chain=prerouting`, **before connection tracking**. Dropping in filter would still create conntrack entries and burn CPU under a flood.

```routeros
/ip firewall filter
add chain=forward connection-state=new action=jump jump-target=detect-ddos
add action=return chain=detect-ddos dst-limit=32,32,src-and-dst-addresses/10s
add action=add-dst-to-address-list address-list=ddos-targets address-list-timeout=10m chain=detect-ddos
add action=add-src-to-address-list address-list=ddos-attackers address-list-timeout=10m chain=detect-ddos
/ip firewall raw
add action=drop chain=prerouting dst-address-list=ddos-targets src-address-list=ddos-attackers
```

How it works: new connections jump to `detect-ddos`; flows under the `dst-limit` rate `return`; the rest add the destination to `ddos-targets` and the source to `ddos-attackers`; raw drops traffic matching both lists.

Pair it with:

```routeros
/ip/settings/set tcp-syncookies=yes
```

Optionally set `/ip/settings/set rp-filter=loose` (or `strict`). Do not use `strict` with policy routing / custom routing tables.

Version notes: 7.17 added a TCP SYN flood warning log and fixed `dst-limit`/`limit` with `burst=0`; 7.20 fixed `dst-limit` rates above `10000`.

## Port Scan Detection — `psd`

`psd` is a RouterOS-specific TCP matcher that scores packets by destination port type:

```routeros
/ip/firewall/filter/add chain=input \
  protocol=tcp psd=21,3s,3,1 \
  action=add-src-to-address-list address-list=port-scanners \
  address-list-timeout=1h comment="dos-detect-portscan"
```

**`psd=weight,delay,low-port-weight,high-port-weight` tuple:**
- `weight` — total score threshold that triggers the rule (21 in example)
- `delay` — max time between probe packets to count as one scan (3s)
- `low-port-weight` — score added per probe of a port < 1024 (3 pts in example)
- `high-port-weight` — score added per probe of a port ≥ 1024 (1 pt in example)

**The last two values are weight scores, NOT port ranges.** Common mistake: treating them as `low-port,high-port` boundaries.

Example: `psd=21,3s,3,1` — scanning 7 low ports (7×3=21) triggers the rule within 3 seconds.

## `action=tarpit`

TCP tarpit — completes the handshake but stalls the connection indefinitely:

```routeros
/ip/firewall/filter/add chain=input \
  src-address-list=dos-attackers \
  connection-limit=3,32 protocol=tcp \
  action=tarpit comment="dos-tarpit-blacklist"
```

**How it works:** Completes SYN/SYN-ACK/ACK, then advertises a TCP receive window size of **zero**. The sender cannot transmit data (zero window = no send permission) and the connection stalls in ESTABLISHED state indefinitely, consuming attacker resources.

- TCP only (`protocol=tcp` required)
- Does NOT throttle bandwidth — data never flows at all
- Effective against connection-flood attacks where cost is per-connection

## `connection-limit` Tuple

```routeros
/ip/firewall/filter/add chain=input \
  protocol=tcp connection-state=new connection-limit=100,32 \
  action=add-src-to-address-list address-list=dos-attackers \
  address-list-timeout=1h comment="dos-detect-ddos"
```

**`connection-limit=X,Y` tuple:**
- `X` — maximum simultaneous connections
- `Y` — netmask prefix bits (32 = per individual IP, 24 = per /24 subnet)

`connection-limit=100,32` = max 100 simultaneous connections per individual IP.

Add `connection-state=new` guard — limits evaluation to new connections only, reducing router CPU load under the attack conditions the rule is meant to detect.

## Combined Blacklist Pattern

Three-stage pattern: detect → blacklist → tarpit:

```routeros
# Stage 1: detect port scanners
/ip/firewall/filter/add chain=input \
  protocol=tcp psd=21,3s,3,1 \
  action=add-src-to-address-list address-list=port-scanners \
  address-list-timeout=1h comment="dos-detect-portscan"

# Stage 2: detect connection floods
/ip/firewall/filter/add chain=input \
  protocol=tcp connection-state=new connection-limit=100,32 \
  action=add-src-to-address-list address-list=dos-attackers \
  address-list-timeout=1h comment="dos-detect-ddos"

# Stage 3: tarpit blacklisted sources (connection-limit=3,32 is the second gate)
/ip/firewall/filter/add chain=input \
  src-address-list=dos-attackers connection-limit=3,32 protocol=tcp \
  action=tarpit comment="dos-tarpit-blacklist"
```

Note: the tarpit rule also requires `connection-limit=3,32` — blacklisted IPs with fewer than 3 simultaneous connections are not tarpitted by this pattern.

The `address-list-timeout=1h` on detection rules means entries auto-expire — no manual cleanup needed.

For flood-scale attacks, prefer the raw-table drop from the official pattern above over the filter-only `tarpit`: filter rules run after connection tracking, raw runs before it.

## Sources

- [DDoS Protection](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/user-guides/ddos-protection)
- [Raw](https://manual.mikrotik.com/docs/firewall-and-quality-of-service/firewall/raw)
- [/ip/settings](https://manual.mikrotik.com/docs/cli-reference/ip/settings)
