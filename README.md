<img src="assets/icon.svg" width="64" height="64" alt="">

# router-os-mikrotik-skill

Agent skills for MikroTik RouterOS v7. Each skill is a folder with a `SKILL.md` and optional
`references/`, in the Agent Skills format used by Claude Code. The agent loads a skill when the task matches its description and gets
correct v7 menu paths, CLI and REST syntax, safe change workflows and the gotchas that usually
cost a lockout or an afternoon. Content is checked against RouterOS **7.25.2 stable** and
**7.24.7 long-term** (November 2026), using the official MikroTik documentation and changelogs.

## Why these skills

- **Current.** Written against 7.25.2, not against what a model remembers from v6 forums.
  Features newer than 7.12 carry the version that introduced them, and each skill has a
  table of renamed and removed menus, so the agent stops suggesting `/ip route rule` or
  `wifiwave2`.
- **Sourced, not guessed.** Every fact traces back to manual.mikrotik.com or a changelog
  entry, and the facts were checked a second time by an independent pass. Where the docs are
  silent or contradict each other, the skill says so instead of inventing a parameter.
- **Fixes real mistakes.** Some of the errors corrected here: `streaming-server=IP:port`
  (invalid; the port is a separate parameter), RADIUS CoA on port 1700 (it is 3799), hotspot
  "RADIUS overrides local users" (local users are checked first), connected-route scope 0
  (it is 10, target-scope 5), and a scheduler `interval=1d` combined with `days=` (not
  supported).
- **Covers what breaks networks.** Five new skills cover bridge VLANs, Wi-Fi, routing, VPN and
  maintenance. These are the areas where a wrong command locks you out, and the skills
  include Safe Mode workflows for remote changes.
- **Light on context.** Each `SKILL.md` stays under 400 lines. Long tables and examples live in
  `references/` and are loaded only when the task needs them.

## Skills

| Area | Skill | Purpose |
| --- | --- | --- |
| Core | [routeros-fundamentals](skills/routeros-fundamentals) | CLI path syntax, REST and binary API, `/ip/service`, users, versions and channels, packages, device-mode, licensing |
| | [routeros-scripting](skills/routeros-scripting) | `.rsc` scripting, `[find]` selectors, `print as-value`, `:onerror`/`:retry`, scheduler and Netwatch snippets |
| | [routeros-command-tree](skills/routeros-command-tree) | `/console/inspect` introspection, CLI-to-REST verb mapping, schema generation |
| | [routeros-maintenance](skills/routeros-maintenance) | Upgrades and firmware, backup vs `/export`, Safe Mode, syslog, NTP, certificates and ACME, DDNS |
| Switching and Wi-Fi | [routeros-bridge-vlan](skills/routeros-bridge-vlan) | Bridge VLAN filtering, switch-chip offload, STP/MSTP, IGMP snooping, port isolation, L3HW |
| | [routeros-wifi](skills/routeros-wifi) | `/interface/wifi` drivers, legacy wireless, WPA3 and roaming, VLANs on wifi, both CAPsMAN systems |
| Routing and security | [routeros-routing](skills/routeros-routing) | Static and policy routing, WAN failover, ECMP, VRF, v7 routing filters, OSPF, BGP, BFD |
| | [routeros-firewall](skills/routeros-firewall) | Filter, NAT, mangle, raw, conntrack, FastTrack, address/interface lists, DDoS protection |
| | [routeros-vpn](skills/routeros-vpn) | WireGuard, IPsec IKEv2 and xfrm, OpenVPN, L2TP, SSTP, ZeroTier, Back To Home |
| | [routeros-hotspot](skills/routeros-hotspot) | Captive portal, walled garden, RFC 8910 API, RADIUS/RadSec, User Manager |
| Containers | [routeros-container](skills/routeros-container) | `/container`: device-mode, VETH networking, images, env/mounts, limits, healthchecks |
| | [routeros-app-yaml](skills/routeros-app-yaml) | `/app` built-in and custom YAML apps, placeholders, secrets, validation errors |
| Lab and provisioning | [routeros-qemu-chr](skills/routeros-qemu-chr) | CHR under QEMU: images, VirtIO, SeaBIOS vs UEFI, port forwarding, CI |
| | [routeros-netinstall](skills/routeros-netinstall) | `netinstall-cli`, Windows GUI and the `/tool/netinstall` package |
| Protocols and debugging | [routeros-sniffer](skills/routeros-sniffer) | `/tool/sniffer`, TZSP streaming, `sniff-tzsp` mangle, torch, port mirroring |
| | [routeros-mndp](skills/routeros-mndp) | MNDP wire format, `/ip/neighbor`, CDP/LLDP discovery settings |
| | [routeros-mac-telnet](skills/routeros-mac-telnet) | MAC-Telnet protocol, MD5 and EC-SRP auth, `/tool/mac-server` |

## Install

Clone the repository and link the skills into your personal skills directory:

```sh
git clone https://github.com/yukij3/router-os-mikrotik-skill.git
cd router-os-mikrotik-skill
mkdir -p ~/.claude/skills && for d in "$PWD"/skills/routeros-*; do ln -sfn "$d" ~/.claude/skills/; done
```

Use `cp -R` instead of `ln -sfn` if you prefer a copy. For a single project, put the folders in
`.claude/skills/` inside that project. Other tools that read the Agent Skills format can use the
same folders as-is.

## Scope

- RouterOS 7.x only. RouterOS v6 is not covered; answers about v6 will be unreliable.
- Treat generated configuration as a draft. Test it on a lab router or a
  [CHR](skills/routeros-qemu-chr) first, and use Safe Mode (`Ctrl+X`) on remote devices.

## Credits

Twelve of the skills started as forks of [tikoci/routeros-skills](https://github.com/tikoci/routeros-skills)
and have since been revised for 7.24/7.25. `routeros-bridge-vlan`, `routeros-wifi`,
`routeros-routing`, `routeros-vpn` and `routeros-maintenance` are new.

## License

[MIT](LICENSE). Copyright (c) 2026 tikoci and dmitrijtretakov.
