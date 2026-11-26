# /ip/neighbor/discovery-settings: LLDP, LLDP-MED, CDP and PoE

Detail for the `routeros-mndp` skill. Verified against RouterOS 7.25.2 stable /
7.24.7 long-term, 2026-11-26. MNDP shares these settings with CDP and LLDP.

## General properties

| Property | Values / default | Since | Notes |
|----------|------------------|-------|-------|
| `discover-interface-list` | interface list; default `static` | — | Static (non-dynamic) interfaces only. Loopback, LTE and `ppp-out` are always excluded since 7.20 |
| `protocol` | `cdp`, `lldp`, `mndp` (combinable) | — | Which protocols are sent and received |
| `mode` | `rx-only`, `tx-only`, `tx-and-rx`; default `tx-and-rx` | 7.7 | |
| `discover-interval` | `5s..9h6m8s`; default `30s` | 7.16 | Was a fixed 60s before 7.16. CDP/LLDP TTL TLV = `discover-interval * 4 + 1` |
| `add-dns-entries` | `yes`/`no` | 7.23 | Creates dynamic (`D`) `/ip/dns/static` entries for discovered neighbors |
| `add-dns-entries-suffix` | — | 7.23 | Domain suffix for those entries. Type and default are not documented yet |

```routeros
/ip/neighbor/discovery-settings/set add-dns-entries=yes
```

## LLDP TLV options

| Property | Default | Since | What it advertises |
|----------|---------|-------|--------------------|
| `lldp-mac-phy-config` | `no` | 7.15 | MAC/PHY Configuration/Status TLV. Sent automatically whenever LLDP-MED is active, even if `no` |
| `lldp-max-frame-size` | `no` | 7.15 | Maximum Frame Size TLV (value = `l2mtu + 18`) |
| `lldp-vlan-info` | `no` | 7.16 | On vlan-filtering bridge ports: Port VLAN ID (bridge port `pvid`), Port And Protocol VLAN ID (always "not supported"/"not enabled"), VLAN Name (up to 10 active VLANs from `/interface/bridge/vlan`) |
| `lldp-dcbx` | `no` | 7.17 | Marvell Prestera switches only (e.g. CRS3xx): ETS Configuration, ETS Recommendation, PFC Configuration (willingness bit disabled), Application Priority, Application VLAN (empty table) |
| `lldp-poe-power` | `yes` | 7.15 | PoE power TLV. Replaces `poe-lldp-enabled` formerly under `/interface/ethernet/poe` |
| `lldp-med` | `yes` | 7.23 | LLDP-MED Media Capabilities TLV. Must stay enabled when `lldp-med-net-policy-vlan` is used |
| `lldp-med-net-policy-vlan` | `disabled` (`0..4094`) | — | LLDP-MED network policy: Voice, Tagged, L2 priority 0, DSCP 0 |

Other LLDP behavior by version:

- **7.15:** Port Description TLV (e.g. `bridge/ether1`); Port ID advertises the
  physical interface name; LLDP-MED Power TLV is always sent if MED was received.
- **7.16:** LLDP-MED timeout added.
- **7.17:** neighbor `platform`, `version` and board are parsed from the LLDP
  System Description.
- **7.18:** IPv6 capability advertised based on the global "disable IPv6"
  setting; actual PSE power pair reported; `power-via-mdi-short` used only for
  PSE Type 1 (802.3af).
- **7.20:** LLDP Power via MDI TLV carries 802.3bt fields.
- **7.23:** advertised `interface-name` updates dynamically; LLDP MAC/PHY TLV
  fixed; new read-only `/ip/neighbor/lldp` menu (WinBox got a filtered LLDP tab
  in 7.25). Its exact property set is not documented yet.
- **7.24:** dying gasp. A TTL=0 LLDP/MNDP/CDP packet is sent before a graceful
  reboot, shutdown or upgrade, and receivers drop the neighbor.
- **7.25:** regression where LLDP neighbors lost `version` and board; fixed in
  **7.25.2**. On 7.25 / 7.25.1, missing LLDP version/board fields come from
  this bug.

## CDP

- **7.25.2:** CDP version 2 frames are processed.

## LLDP-MED on a bridge

When `lldp-med-net-policy-vlan` is used on a bridge:

- (R/M)STP must be enabled on the bridge (`protocol-mode`).
- Leave CDP out of `protocol=` to avoid LLDP-MED misconfiguration.
- `lldp-med` must be `yes` (7.23+).

```routeros
/ip/neighbor/discovery-settings/set protocol=lldp,mndp
```

## Removed / deprecated

| Item | Change |
|------|--------|
| `/interface/ethernet/poe` `poe-lldp-enabled` | Moved to `/ip/neighbor/discovery-settings` `lldp-poe-power` (7.15) |
| Fixed 60s announce interval | Replaced by `discover-interval`, default 30s (7.16) |
| `reset` command in `discovery-settings` | Removed (7.21.4 / 7.22.1) |
