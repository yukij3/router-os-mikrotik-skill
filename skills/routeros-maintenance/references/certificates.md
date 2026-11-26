# Certificates (`/certificate`)

Referenced from `routeros-maintenance` SKILL.md.

## ACME / Let's Encrypt

```routeros
/certificate/add-acme domain-names=[/ip/cloud/get dns-name]
/certificate/acme-renew
```

| Topic | Notes |
|---|---|
| Command | `/certificate/add-acme`; `enable-ssl-certificate` is obsolete and returns an error |
| CA | `directory-url` defaults to Let's Encrypt (7.23+) |
| Renewal | Automatic at 80 % of validity; manual `/certificate/acme-renew` (7.24+) |
| IP Cloud name `*.sn.mynetname.net` (7.16+) | DNS-01 through MikroTik Cloud, no inbound port 80. Wildcard `*.<id>.sn.mynetname.net` works (fixed 7.25) |
| Own domain | HTTP-01: enable `www` in `/ip/service` and make TCP 80 reachable from the internet |

Docs: <https://manual.mikrotik.com/docs/cli-reference/certificate/add-acme>

## Built-in Trust Store

- `/certificate/builtin` (7.19+) lists bundled root CAs (`Amazon Root CA 1` added 7.20).
- `/certificate/settings builtin-trust-store=` (replaced `builtin-trust-anchors` in 7.21); value `default` since 7.22.1. Per-certificate `trust-store` since 7.21.
- With `default`, these clients trust public servers without importing CAs: `fetch`, `mqtt`, `email`, `netwatch`, `container`, `lora`, `wiliot`, `dns`, `www`, `reverse-proxy`.
- SMIPS devices carry a reduced store: ISRG Root X1/X2/YE/YR and DigiCert Global Root G2.

## Own CA, Import and Export

- Certificate templates created with `add` and then `sign` keep their name after signing.
- **Removing a CA removes every certificate it issued.**
- `/certificate/import` deletes the uploaded file from `/file` after importing.
- `no-key-export=yes` (7.16+) prevents exporting the private key.
- PKCS#12 export uses AES encryption (7.24+).
- `/export` does not include certificates; back them up with `/certificate/export-certificate`.

Docs: <https://manual.mikrotik.com/docs/authentication-authorization-accounting/certificates>
