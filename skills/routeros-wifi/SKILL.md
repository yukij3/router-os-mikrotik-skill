---
name: routeros-wifi
description: "RouterOS v7 wireless: the /interface/wifi menu (wifi-qcom, wifi-qcom-ac, wifi-qcom-be, wifi-mediatek drivers), legacy /interface/wireless, and both CAPsMAN systems. Use when: choosing the right wireless package for a MikroTik board, configuring an AP or station, WPA3/OWE/PPSK security, 802.11r/k/v roaming and band steering, Wi-Fi 7 MLO, VLANs on wifi (datapath vlan-id), setting up CAPsMAN and CAPs, migrating from wifiwave2 or legacy wireless, or when the user mentions /interface/wifi, /interface/wifi/network, /interface/wifi/capsman, /interface/wifi/provisioning, /caps-man, /interface/wireless, wifiwave2, datapath, multi-passphrase, hAP ax, hAP be, cAP ax or CAPsMAN on MikroTik."
---

# RouterOS WiFi & CAPsMAN

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26. Features newer than 7.12 carry their minimum version inline.

Package install, CLI syntax and REST basics are in `routeros-fundamentals`. Bridge/VLAN mechanics (pvid, `/interface/bridge/vlan`) are in `routeros-bridge-vlan`.

## Two Wireless Stacks

| | New WiFi | Legacy |
|---|---|---|
| Menu | `/interface/wifi` | `/interface/wireless` (and `/interface/w60g` for 60 GHz) |
| Packages | `wifi-qcom`, `wifi-qcom-ac`, `wifi-qcom-be`, `wifi-mediatek` | `wireless` |
| Controller | `/interface/wifi/capsman` + `/interface/wifi/provisioning` | `/caps-man/...` |
| CAP side | `/interface/wifi/cap` | `/interface/wireless/cap` |

The two stacks never manage each other's radios. Both controllers can run on the same router at the same time.

## Which Package for Which Board

| Device family | Menu | Package | Notes |
|---|---|---|---|
| Wi-Fi 6/6E Qualcomm (hAP ax², hAP ax³, cAP ax, wAP ax, Chateau ax, L11/L22) | `/interface/wifi` | `wifi-qcom` | Legacy `wireless` cannot drive the built-in radios |
| Wi-Fi 7 Qualcomm (e.g. hAP be³ Media) | `/interface/wifi` | `wifi-qcom-be` | MLO, 320 MHz, `preamble-puncturing` (7.24+) |
| Wi-Fi 6/7 MediaTek (e.g. hAP ax S, hAP be lite) | `/interface/wifi` | `wifi-mediatek` | 7.19+; supports meshpoint (802.11s) |
| Wi-Fi 5 ARM (Audience, Chateau D53, hAP ac², hAP ac³, cAP ac, cAP XL ac, wAP ac `RBwAPG-5HacD2HnD`, LDF 5 ac, LHG XL 5/52 ac, NetMetal ac², mANTBox 52 15s, SXTsq 5 ac) | `/interface/wifi` **or** `/interface/wireless` | `wifi-qcom-ac` **or** `wireless` | Pick one, see trade-offs below |
| 802.11a/b/g/n, MIPS 802.11ac (e.g. hAP ac, RB4011 WiFi), 60 GHz | `/interface/wireless`, `/interface/w60g` | `wireless` | Legacy only |
| Controller with no local radios | `/interface/wifi/capsman` and/or `/caps-man` | base `routeros` (new CAPsMAN) / `wireless` (legacy CAPsMAN) | New CAPsMAN is in the base bundle since 7.13 |

**`wifi-qcom-ac` vs `wireless` on Wi-Fi 5 ARM boards:**
- `wifi-qcom-ac` fits 16 MB flash (7.13+) and adds WPA3, OWE, 802.11r/k/v, but uses more RAM.
- `wifi-qcom-ac` lacks: `distance` beyond ~2 km, `datapath.vlan-id`, Nstreme/Nv2, `station-bridge` to legacy APs, `qos-classifier=dscp-high-3-bits`, `ft-preserve-vlanid`.
- Not compatible with `wifi-qcom-ac`: `RB4011iGS+5HacQ2HnD-IN` (2.4 GHz unsupported), `Cube 60Pro ac`, `RBwAPG-5HacT2HnD`.

Driver packages are mutually exclusive. Since **7.18**, installing more than one at a time is blocked.

Caveat: `wifi-mediatek` first appears in 7.19. The exact first release of `wifi-qcom-be` was not pinned down; check the changelog for the board before assuming support on an older version.

## Configuration Model

`/interface/wifi` reads settings from profiles: `aaa`, `channel`, `configuration` (can reference the others), `datapath`, `interworking`, `security`, `steering`, plus `access-list` and `security/multi-passphrase`.

**Precedence (highest first):**
1. Inline property on `/interface/wifi` (e.g. `channel.width=...`)
2. Profile referenced directly on `/interface/wifi` (e.g. `channel=ch5g`)
3. Inline property inside the `/interface/wifi/configuration` profile
4. Profile referenced inside the `/interface/wifi/configuration` profile

Inspect: `/interface/wifi/print detail` shows the effective config; `/interface/wifi/print detail config` (7.15+) shows only what is set directly on the interface.

### Minimal standalone AP

```routeros
/interface/wifi/security/add name=sec1 authentication-types=wpa2-psk,wpa3-psk passphrase="change-me-please"
/interface/wifi/configuration/add name=cfg1 ssid=MyNet country=Latvia security=sec1
/interface/wifi/set [find default-name=wifi1] configuration=cfg1 disabled=no
```

Set `country` to the real country of operation. Since **7.15** wifi interfaces are disabled by default, so `disabled=no` is required.

### `/interface/wifi/network` (7.22+)

A higher-level, declarative menu that bundles SSID, security, datapath and MLO (`mlo=yes`, `labels=all|2ghz|5ghz|6ghz|<MAC>`). Per-radio `country` / `channel` / `extra-labels` go in `/interface/wifi/network/radio`.

A radio gets its config from the first match of:
1. A static `/interface/wifi` entry for that radio MAC
2. `/interface/wifi/provisioning`
3. `/interface/wifi/network` (interface flag `N`)
4. A disabled stub

`/interface/wifi/liberate` detaches a network-managed interface so you can edit it directly.

## Key Properties

**Channel** (`/interface/wifi/channel` or `channel.*`): `band`, `width` (`20mhz`, `20/40mhz`, `20/40/80mhz`, `20/40/80+80mhz`, `20/40/80/160mhz`, `20/40/80/160/320mhz`), `frequency` (single, list, or range like `5180-5240:20`), `secondary-frequency`, `skip-dfs-channels=disabled|all|10min-cac`, `reselect-interval` (7.15+), `reselect-time` (7.19+), `deprioritize-unii-3-4` (7.20+), `preamble-puncturing` (802.11be, 7.24+).

**Configuration** (`/interface/wifi/configuration` or `configuration.*`):

| Property | Notes |
|---|---|
| `mode` | `ap`, `station`, `station-bridge`, `station-pseudobridge`, `meshpoint` |
| `country` | Default `Latvia` since 7.14 |
| `installation` | `indoor`/`outdoor` (7.17+); documented default `indoor`, defconf sets `any` |
| `distance` | Multi-km on `wifi-qcom` (7.14+) and `wifi-qcom-be` (7.25+); not on `wifi-qcom-ac` |
| `dtim-period` | defconf uses `3` since 7.24 |
| `multicast-enhance` | `enabled`/`disabled` |
| `qos-classifier` | `priority` / `dscp-high-3-bits` (latter not on `wifi-qcom-ac`) |
| `station-roaming` | `yes`/`no` (7.17+) |
| `max-clients` | 7.18+ |
| `hide-ssid` | Hidden SSID |
| `manager` | `capsman` / `capsman-or-local`, set **only on a CAP's interface** |

Security, WPA3, OWE, PPSK, 802.11r and steering: [security-roaming.md](./references/security-roaming.md).

## CAPsMAN (summary)

```routeros
# Controller
/interface/wifi/capsman/set ca-certificate=auto enabled=yes
# plus /interface/wifi/security, /datapath, /configuration profiles and a
# /interface/wifi/provisioning rule (action=create-dynamic-enabled ...), see capsman.md

# CAP
/interface/wifi/cap/set discovery-interfaces=bridgeLocal enabled=yes slaves-datapath=capdp
/interface/wifi/set [find default-name=wifi1] configuration.manager=capsman datapath=capdp disabled=no
```

- `traffic-processing=on-cap` (default, local forwarding), `on-capsman` / `on-capsman-secure` (**7.21+**, tunnelled to the controller; not on `wifi-qcom-ac`, not with MLO).
- Full setup, discovery and VLAN handling (including the `wifi-qcom-ac` workaround): [capsman.md](./references/capsman.md).

## Gotchas

- **`configuration.manager=capsman` goes on the CAP's `/interface/wifi`, never in a controller-side configuration profile.** CAPsMAN cannot manage its own local radios via `manager=capsman`.
- **Upgrading from <7.12 to 7.13+** with `check-for-updates` must step through **7.12** so wireless packages convert automatically.
- **`station-bridge` only works wifi-to-wifi or wireless-to-wireless**, never across stacks.
- **802.11r (`ft=yes`) requires all APs managed by one RouterOS instance** (local, or the same CAPsMAN) and does not work with OWE.
- **PPSK (`multi-passphrase`) is incompatible with `wpa3-psk`.**
- **`wifi-qcom-ac` + `interface=all` bridge port:** a wildcard bridge port prevents setting per-interface `pvid` on dynamic wifi interfaces. Add ports individually and use `slaves-static=yes` on the CAP. `access-list` VLAN assignment on `wifi-qcom-ac` also needs a `pvid` on the interface.
- **Defaults changed:** `country=Latvia` (7.14), interfaces disabled by default (7.15). Scripts written for 7.13 that rely on defaults may leave radios off or on the wrong regulatory domain.

## Deprecated / Renamed

| Item | Change | Version |
|---|---|---|
| `wifiwave2` package, `/interface/wifiwave2` | Split into `wifi-qcom` / `wifi-qcom-ac`, menu moved to `/interface/wifi` | 7.13 |
| `wireless` in the `routeros` bundle | Now a standalone `wireless.npk` | 7.13 |
| `openflow-switch` | Removed | 7.13 |
| `available-channels` | Renamed `channel-priorities` | 7.15 |
| `actual-configuration` | Replaced by `print` / `print config` | 7.15 |
| `ft-wpa2-eap` | Renamed `ft-eap` | 7.21 |

## References

- [security-roaming.md](./references/security-roaming.md): authentication types, WPA3/SAE, OWE transition, PPSK, 802.11r, 802.11k/v steering
- [capsman.md](./references/capsman.md): new CAPsMAN setup, provisioning, discovery, datapath/VLANs, legacy `/caps-man`
