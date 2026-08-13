# Vaultwarden

## Summary

Vaultwarden is the homelab's password manager: a lightweight reimplementation of
the Bitwarden server, compatible with the official Bitwarden clients and browser
extensions.

It runs in Docker on `atlas` and is intended for LAN and VPN access only. There
is no public exposure.

Access is through `https://pass.home.arpa`, proxied by `rpi-01` with a
certificate signed by the homelab's own certificate authority. Remote access is
over WireGuard; the service is never published to the internet.

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

Self-signed certificates clear that bar in a browser, where an exception can be
stored, but the Bitwarden mobile applications reject them outright and offer no
way to add an exception. Making the mobile clients usable therefore required a
certificate signed by an authority the device already trusts.

### Why not a public certificate

No public CA can issue for `home.arpa`. It is a reserved special-use domain
(RFC 8375), and CA/Browser Forum rules have prohibited certificates for internal
names since 2015 — nobody can demonstrate control over a name that belongs to
every network at once.

That restriction binds public CAs, not the operator of the network. An internal
CA can sign for `home.arpa`, and the resulting certificate is cryptographically
identical to a public one. What differs is that the trust is installed on the
devices rather than shipped with them.

The alternative considered was registering a public domain and issuing through
DNS-01, which would work on every device without installing anything. It was
rejected because it means depending on an external registrar and CA for a service
that never leaves the LAN.

### Current arrangement

```
client → https://pass.home.arpa        (resolved by AdGuard to 192.168.1.30)
       → nginx on rpi-01                certificate signed by the internal CA
       → https://192.168.1.22:8222      Vaultwarden's own self-signed cert
```

`rpi-01` terminates the trusted certificate and proxies onward with
`proxy_ssl_verify off`, because the backend certificate is not signed by the
internal CA. The hop stays encrypted; only its authenticity is unverified, over a
LAN segment where the alternative was plaintext.

Vaultwarden keeps `ROCKET_TLS` and its own self-signed certificate, which is what
secures that final hop.

The CA lives on `rpi-01`. See [`docs/nodes/rpi-01.md`](../nodes/rpi-01.md#internal-certificate-authority)
for the certificate parameters and renewal.

### Installing trust on a device

The root certificate is published at `http://ca.home.arpa/`, reachable from the
LAN and over WireGuard. It is served over HTTP deliberately: a device that does
not yet trust the CA cannot validate a certificate signed by it, so requiring
HTTPS to obtain the CA would be circular. What is published is the public half.

| Platform | Procedure |
|---|---|
| Android | Settings → Security → Encryption & credentials → Install a certificate → **CA certificate** |
| iOS | Install the profile, then Settings → General → About → **Certificate Trust Settings** and enable full trust. The second step is mandatory and easily missed |
| Windows | `certutil -addstore -f Root homelab-ca.crt` as administrator |
| Firefox | Uses its own trust store: import separately, or set `security.enterprise_roots.enabled` |

Every new device needs this once. That recurring cost is the price of not
depending on a public CA.

### Outstanding

Since Android API 24, applications do not trust user-installed CAs unless they
declare it in their network security configuration. **Whether the Bitwarden
Android application does so has not yet been verified on a real device.** Until
that test is performed, mobile support is expected rather than confirmed.

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
