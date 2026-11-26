# CAPsMAN: New WiFi and Legacy

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term, 2026-11-26.

## Two separate controllers

| | New WiFi CAPsMAN | Legacy CAPsMAN |
|---|---|---|
| Controller menus | `/interface/wifi/capsman`, `/interface/wifi/provisioning`, `/interface/wifi/configuration`, `/interface/wifi/datapath` | `/caps-man/manager`, `/caps-man/provisioning`, `/caps-man/configuration`, `/caps-man/datapath` |
| CAP menu | `/interface/wifi/cap` | `/interface/wireless/cap` |
| Manages | `wifi-qcom`, `wifi-qcom-be`, `wifi-mediatek`, `wifi-qcom-ac` CAPs | `/interface/wireless` CAPs |
| Package on controller | base `routeros` (since 7.13) | `wireless` |

Neither can manage the other's radios. Both can run on one router.

## New CAPsMAN setup

**Controller:**
1. Create `/interface/wifi/security`, `/interface/wifi/datapath` and `/interface/wifi/configuration` profiles.
2. Add `/interface/wifi/provisioning` rules:
   - `action=create-dynamic-enabled|create-enabled|create-disabled`
   - match with `supported-bands`, `supported-hw-caps` (7.22+)
   - naming with `name-format`, `slave-name-format`
   - `multi-link-mode` for MLO
3. Enable the controller:

```routeros
/interface/wifi/capsman/set ca-certificate=auto enabled=yes
```

**CAP:**

```routeros
/interface/wifi/cap/set discovery-interfaces=bridgeLocal enabled=yes slaves-datapath=capdp
/interface/wifi/set [find default-name=wifi1] configuration.manager=capsman datapath=capdp disabled=no
```

`configuration.manager=capsman` (or `capsman-or-local`) belongs on the CAP's interface only. Never put it in a controller-side configuration profile. The controller cannot manage its own local radios via `manager=capsman`.

## Discovery

A CAP finds the controller via:
- L2 broadcast on `discovery-interfaces`
- DHCP option 138
- DHCP option 15 domain, looked up as `_capsman._tcp.<domain>`, or `_capsman._tcp.lan`
- Explicit `caps-man-addresses`: IP, IPv6 link-local (7.17+), hostname (7.22+)

## Datapath and forwarding

`traffic-processing`:
- `on-cap` (default): CAP bridges client traffic locally.
- `on-capsman` (**7.21+**): traffic is tunnelled to the controller.
- `on-capsman-secure` (**7.21+**): encrypted tunnel.

`on-capsman` modes are not supported on `wifi-qcom-ac` or with MLO.

## VLANs on CAPs

**`wifi-qcom`, `wifi-qcom-be`, `wifi-mediatek`:** set `vlan-id` in the datapath profile. The CAP tags client traffic itself.

**`wifi-qcom-ac`:** `datapath.vlan-id` is not supported. Instead:
1. Leave `vlan-id` out of the controller's datapath.
2. On the CAP, `/interface/wifi/cap/set slaves-static=yes` so wifi interfaces are static and can be bridge ports.
3. On the CAP's bridge, add each wifi interface as its own port with a `pvid`, and enable `vlan-filtering=yes` (see `routeros-bridge-vlan`).
4. Do not use a wildcard `interface=all` bridge port: it blocks per-interface `pvid` on wifi interfaces.

`access-list` VLAN assignment on `wifi-qcom-ac` also only works when the interface has a `pvid`.
