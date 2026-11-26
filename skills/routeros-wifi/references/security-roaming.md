# WiFi Security, Roaming & Steering (`/interface/wifi`)

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.

## Authentication types (`/interface/wifi/security`)

`authentication-types` accepts a list of: `wpa-psk`, `wpa2-psk`, `wpa2-psk-sha2` (7.19+), `wpa-eap`, `wpa2-eap`, `wpa3-psk`, `wpa3-psk-gd`, `owe`, `wpa3-eap`, `wpa3-eap-192`.

Defaults when a passphrase is set and no type is given:

| Radio | Default |
|---|---|
| Wi-Fi 5/6, 2.4/5 GHz | `wpa2-psk` |
| Wi-Fi 7, 2.4/5 GHz | `wpa2-psk,wpa3-psk,wpa3-psk-gd` |
| 6 GHz | `wpa3-psk,wpa3-psk-gd` |

## WPA3 / SAE

| Property | Values / default |
|---|---|
| `management-protection` | `disabled`/`allowed`/`required` (WPA3 needs it) |
| `beacon-protection` | `disabled`/`enabled` (enabled by default on 802.11be) |
| `dh-groups` | `19,20,21` |
| `sae-anti-clogging-threshold` | default `5` |
| `sae-max-failure-rate` | default `40` |
| `sae-pwe` | `both`/`hash-to-element`/`hunting-and-pecking` |

WPA3-PSK does not enforce an 8-character minimum passphrase.

## OWE and OWE transition mode

Pair an open AP with a hidden OWE AP:

- Open (visible) interface: `security.authentication-types=""` `security.owe-transition-interface=auto`
- OWE (hidden) interface: `security.authentication-types=owe` `security.owe-transition-interface=auto` `configuration.hide-ssid=yes`

802.11r (`ft=yes`) does not work with OWE.

## PPSK: `/interface/wifi/security/multi-passphrase` (7.17+)

- Up to 10,000 passphrases, shared across interfaces.
- Per-passphrase `vlan-id`, `expires`, `isolation`.
- **Incompatible with `wpa3-psk`.**

## Fast Transition (802.11r)

Security properties: `ft=yes`, `ft-over-ds=yes|no`, `ft-mobility-domain` (default `0xADC4`), `ft-preserve-vlanid=yes|no` (default `yes`, not on `wifi-qcom-ac`).

All APs in the mobility domain must be managed by the **same RouterOS instance** (one router's local radios, or one CAPsMAN). `ft-wpa2-eap` was renamed `ft-eap` in 7.21.

## Steering (`/interface/wifi/steering`)

| Property | Default | Notes |
|---|---|---|
| `rrm` | `yes` | 802.11k neighbor reports |
| `wnm` | `yes` | 802.11v solicited BSS transition |
| `neighbor-group` | dynamic | Auto-grouped by matching SSID + auth, unless EAP is used |
| `2g-probe-delay` | | 7.18+. Delays the first 3 2.4 GHz probe responses within 60 s when client SNR > 35 dB and a 5 GHz AP is in the same neighbor group |

**Unsolicited 802.11v BTM (7.21+):**

| Property | Default |
|---|---|
| `transition-threshold` | `-80` (dBm) |
| `transition-threshold-time` | `10s` |
| `transition-request-period` | `30s` |
| `transition-request-count` | `3` |
| `transition-time` | `unlimited` (`unlimited`/`immediate`/`<time>`) |
