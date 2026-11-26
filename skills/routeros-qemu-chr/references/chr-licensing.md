# CHR Licensing

Verified against RouterOS 7.25.2 stable / 7.24.7 long-term (2026-11-26).

## License Tiers

| Level | Speed limit | Cost | Notes |
|-------|-------------|------|-------|
| `free` | 1 Mbit/s upload per interface | $0 | No registration needed. Unlimited use |
| `p1` | 1 Gbit/s upload per interface | paid | Perpetual |
| `p10` | 10 Gbit/s upload per interface | paid | Perpetual |
| `p-unlimited` | Unlimited | paid | Perpetual |
| 60-day trial | Same as the chosen paid level | $0 | Requires a mikrotik.com account. See below |

**`IP/Cloud` requires a paid perpetual license** — it does not work on `free` or trial.

## Free License (Default)

Every CHR instance starts with a free license. No registration, no activation — just boot and use.

**The limit is an upload limit per interface** — traffic sent out of each interface is capped at 1 Mbit/s. It is not a total forwarding or download cap. In practice:
- REST API calls, SSH, WinBox, WebFig — small responses, rarely noticeable
- Packet sniffer TZSP streaming — fine for debugging (TZSP packets are small)
- Forwarded traffic leaving any interface — capped at 1 Mbit/s per interface
- If users report "slow" traffic through a CHR, the free license limit is the most likely cause

**1 Mbit/s is sufficient for:**
- Development and API testing
- Protocol debugging with `/tool/sniffer` or mangle `sniff-tzsp`
- CI/CD integration tests
- Learning RouterOS configuration
- Container (`/container`) development

## 60-Day Trial License

A free 60-day trial lets you test **any paid level** — `p1`, `p10` or `p-unlimited` — selected with `level=`.

### How to Obtain

1. Create a free account at [mikrotik.com](https://www.mikrotik.com) (Account → Register)
2. In the CHR, run: `/system/license/renew account=<email> password=<password> level=p-unlimited` (or `level=p1` / `level=p10`)
3. Or via WebFig: System → License → Renew
4. The trial activates immediately — no reboot needed

### Expiry Behavior (trial or perpetual with `limited-upgrades: yes`)

- The CHR **continues to work at the same tier** — the speed limit of the trial/licensed level stays in place; it does NOT drop back to 1 Mbit/s
- **RouterOS upgrades and package changes are disabled** (enable/disable/install/remove)
- If no license is purchased within **2 months after trial expiration**, the CHR instance is removed from the MikroTik.com account
- An expired instance can then only be licensed with a **Prepaid key** — an existing perpetual license cannot be transferred to it
- Since CHR is a VM, you can also delete it and create a fresh instance with a new trial

### When to Use Trial vs Free

| Scenario | License |
|----------|---------|
| API development / testing | Free (1 Mbit/s is fine) |
| Sniffer / TZSP debugging | Free (packet mirroring is low bandwidth) |
| Throughput testing | Trial (need real speeds) |
| QoS / queue testing | Trial (need measurable bandwidth) |
| Testing `IP/Cloud` | Paid perpetual only |
| CI pipelines | Free (API tests don't need throughput) |
| Demo / training | Free or Trial depending on scenario |

## Checking Current License

```routeros
/system/license/print
```

Via REST API:
```sh
curl -u admin: http://<router-ip>/rest/system/license
```

## Licensing, Cloning and `system-id`

Each CHR VM instance needs its own license. Licenses are tied to the VM's `system-id`, generated on first boot. Cloning a disk image (including cloud images, e.g. Linode) duplicates the `system-id` **and the trial state**.

Run this on the first boot of each clone, **while it is still on the `free` level**, before activating a trial or paid license:

```routeros
/system/license/generate-new-id
```

Since 7.15, `generate-new-id` only works on the `free` license level.

For testing purposes, the free license avoids all these concerns — just create and destroy VMs freely.
