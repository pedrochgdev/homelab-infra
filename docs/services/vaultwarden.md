# Vaultwarden

## Summary

Vaultwarden is the homelab's password manager: a lightweight reimplementation of
the Bitwarden server, compatible with the official Bitwarden clients and browser
extensions.

It runs in Docker on `atlas` and is intended for LAN and VPN access only. There
is no public exposure.

## Host

- Node: `atlas` (`192.168.1.22`)
- Runtime: Docker, alongside Jellyfin
- Version: Vaultwarden 1.37.0, web vault 2026.6.4
- Port: `8222`

`atlas` was chosen because Docker was already running there, it had ample spare
memory and disk, and it is an internal-only node. Placing the vault on `rpi-01`
was rejected because that node terminates public ingress.

## Layout

| Path | Purpose |
|---|---|
| `/srv/vaultwarden/docker-compose.yml` | Service definition |
| `/srv/vaultwarden/data` | SQLite database, RSA signing key, attachments |
| `/srv/vaultwarden/ssl` | TLS certificate and key |

## TLS

The Bitwarden web vault refuses to operate over plain HTTP. This is not only the
browser's secure-context requirement for WebCrypto — the client performs its own
scheme check and rejects `http://` even on `localhost`, so an SSH tunnel is not a
workaround.

Vaultwarden therefore terminates TLS itself through `ROCKET_TLS`, currently using
a self-signed certificate valid for 825 days, with subject alternative names for
`vaultwarden.home.arpa`, `localhost`, `192.168.1.22` and `127.0.0.1`.

Access: `https://192.168.1.22:8222`, accepting the certificate warning once.

### Limitation

Self-signed certificates are workable in a browser, where an exception can be
stored, but the Bitwarden mobile applications reject them outright and offer no
way to add an exception. **Mobile clients cannot be used until a real
certificate is in place.**

The intended fix is a Let's Encrypt certificate issued through the DNS-01
challenge against the existing Cloudflare-hosted domain, with an internal A
record. That yields a publicly valid certificate for an internally reachable
address, without exposing the service.

## Access Control

`SIGNUPS_ALLOWED` is set to `false`. Registration was open only long enough to
create the first account, since an open instance on the LAN lets anyone create
their own vault on it.

## Backups

Vaultwarden holds the most sensitive data in the homelab and lives on a single
unmirrored SSD, so it is backed up from day one.

A nightly job at 02:00 on `atlas`:

1. takes a consistent copy with `sqlite3 .backup` rather than copying the live
   database file, which could otherwise capture a partial write
2. includes `rsa_key.pem`, the key that signs session tokens — without it a
   restored database authenticates nobody
3. includes attachments, sends and `config.json` when present
4. pushes a dated tarball to `/srv/nas/backups/atlas/` on `vault`

The daily restic run on `vault` at 03:30 then sweeps that into the encrypted
repository. See [`docs/runbooks/backups.md`](../runbooks/backups.md).

The push uses a dedicated SSH key authorized on `vault` with port forwarding,
agent forwarding, X11 forwarding and pty allocation all disabled, so it can move
files and nothing else.

## Verification

The first backup was verified by extracting the tarball on `vault` and
confirming a valid SQLite 3 database of 68 pages alongside the RSA key. A full
restore test has not yet been performed.

## Operational Notes

- the container is set to `restart: unless-stopped` and survives host reboots
- the data directory is mode 700
- no admin token is configured, so the `/admin` panel is disabled
