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

`DOMAIN` must be set to `https://pass.home.arpa` and not to the address the
container happens to listen on. It is the origin Vaultwarden considers its own,
and it determines the links in Sends and — the part that bites — **the origin
recorded when a passkey or WebAuthn second factor is registered**. A wrong value
causes no visible problem until a passkey is enrolled and then fails to validate
when connecting by the proper name, at which point the symptom points nowhere
near the cause.

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

### Verified on a real device

This was the open question of the whole design, and it was not a safe
assumption: since Android API 24, applications do not trust user-installed CAs
unless they declare it in their network security configuration, and nothing
guaranteed the Bitwarden client did.

**Confirmed 2026-08-13: with the root certificate installed, the mobile
application synchronises against `https://pass.home.arpa`.** The arrangement
works end to end, and the fallback plan — registering a public domain and issuing
through DNS-01 — is not needed.

Worth keeping in mind that this holds for the client version tested. An
application update that tightened its trust configuration would break access
without any change on the server, and the symptom would be a certificate error
that no amount of server-side debugging explains.

## Access Control

`SIGNUPS_ALLOWED` is set to `false`. Registration was open only long enough to
create the first account, since an open instance on the LAN lets anyone create
their own vault on it.

## No Email

No SMTP is configured, so the instance cannot send anything: no password hints,
no email two-factor, no notifications. This is worth stating plainly because the
failure is silent — the web vault accepts a "send me my hint" request and reports
success, and nothing ever arrives.

The password hint is stored in the `users` table and can be read directly from
the database when needed.

**A hint is not a recovery mechanism.** The master password derives the
encryption key on the client; the server never holds it. There is no reset that
preserves the vault, with database access, admin rights or physical access to the
machine. That property is the reason to run this software, and it is also what
makes losing the password final.

The master password therefore belongs somewhere outside every system it
protects — on paper, alongside the restic repository key. Those two secrets share
a property nothing else in the homelab has: no backup can restore them, because
they are what the backups are encrypted with.

This was learned the direct way. The first account, created 2026-07-24, became
unreachable when its master password was forgotten, and was recreated on
2026-08-13. It cost nothing because the vault was still empty; the same mistake
later would have been unrecoverable.

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
