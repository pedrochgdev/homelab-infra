# rpi-01

## Summary

`rpi-01` is the edge node of the homelab. Despite originally being introduced as
an experimentation platform, it has become one of the most load-bearing nodes in
the environment: it serves internal DNS, terminates the WireGuard VPN, runs the
internal reverse proxy, and hosts the Cloudflare tunnel used to publish one
project externally.

It is still a Raspberry Pi and still useful for experimentation, but it should no
longer be treated as a disposable lab node. Several core network functions depend
on it.

## Role

- primary internal DNS (AdGuard Home)
- WireGuard VPN endpoint
- internal reverse proxy (nginx)
- Cloudflare tunnel host
- remote boot control for the workstation (Wake-on-LAN + PXE boot menu)
- internal certificate authority for `*.home.arpa`
- secondary role: experimentation and future cluster work

## Operating System

- Platform: Ubuntu Server
- Version: `24.04.4`
- Kernel: `6.8.0-1057-raspi` (`aarch64`)

## Hardware

- Model: Raspberry Pi 5
- Memory: 16 GB RAM
- Storage: 512 GB NVMe (`nvme0n1`, ~477 GB usable)

## Service Stack

| Service | Purpose | Listening on |
|---|---|---|
| AdGuard Home | Internal DNS resolution and filtering | `53`, admin UI on `8080` |
| nginx | Internal reverse proxy, TLS termination | `80`, `443`, LAN and VPN only |
| `wg-quick@wg0` | WireGuard VPN endpoint | `wg0`, `10.8.0.1/24` |
| `cloudflared` | Cloudflare tunnel to publish one vhost | outbound only |
| `node_exporter` | Metrics for Prometheus | `9100` |
| dnsmasq | proxyDHCP + TFTP for the workstation's remote boot menu (no DNS, `port=0`) | `67`, `69`, `4011`, `10000-10100/udp` |

The remote boot stack (dnsmasq config, TFTP tree, and the `wake-pc` /
`set-boot` / `boot-pc` scripts in `/usr/local/bin`) is documented in
[`docs/services/remote-boot.md`](../services/remote-boot.md).

## Networking

- LAN address: `192.168.1.30`
- VPN interface: `wg0` at `10.8.0.1/24`
- `wlan0` is present but down; the node runs on wired Ethernet

## Reverse Proxy Configuration

Enabled sites:

- `00-default-deny` — explicit catch-all returning `444` for any request on port
  `80` whose `Host` matches no other vhost
- `homelab` — internal `.home.arpa` hostnames mapped to Proxmox, Grafana, the
  display page, both AdGuard instances, and Jellyfin
- `hannah-ai` — the personal project vhost on port `80`, served through the
  Cloudflare tunnel and proxied to the workstation at `192.168.1.109`
- `internal-tls` — the port `443` catch-all plus `pass.home.arpa`, which proxies
  to Vaultwarden on `atlas`
- `ca-download` — publishes the internal CA's root certificate at
  `ca.home.arpa` so devices can install it

The `443` catch-all uses `ssl_reject_handshake` rather than `return 444`. It
aborts the TLS negotiation before presenting any certificate, so a scanner
reaching the address learns nothing — not even that a certificate for
`*.home.arpa` exists.

**Host-based routing is not an access control.** nginx selects a vhost purely
from the `Host` header, and the catch-all only stops names it does not know. Any
name it does know — `proxmox.home.arpa` is about as guessable as a name gets — is
proxied straight through to the backend. That is why the firewall, and not the
catch-all, is what keeps the proxy off the internet. See
[`docs/network.md`](../network.md#host-firewalls).

Retained in `sites-available` but no longer enabled:

- `hannah` — the retired DuckDNS vhost that previously served the same project
  directly on `443` with a Let's Encrypt certificate. See
  [`docs/network.md`](../network.md#retired-exposure) for why it was withdrawn.

The full internal hostname table lives in
[`docs/network.md`](../network.md#internal-reverse-proxy). External hostnames are
deliberately not recorded in this repository.

## Internal Certificate Authority

`rpi-01` runs the homelab's own certificate authority, which exists because no
public CA may issue for `home.arpa`. The reasoning is recorded in
[`docs/services/vaultwarden.md`](../services/vaultwarden.md#tls), the service
that motivated it.

| Path | Contents |
|---|---|
| `/etc/homelab-ca/ca.key` | Root private key, RSA 4096, mode 600 |
| `/etc/homelab-ca/ca.crt` | Root certificate, valid 10 years |
| `/etc/homelab-ca/certs/internal.key` | Leaf private key |
| `/etc/homelab-ca/certs/fullchain.crt` | Leaf plus root, what nginx serves |
| `/usr/local/bin/renew-internal-cert.sh` | Reissues the leaf when it is close to expiry |
| `internal-cert-renew.timer` | Weekly check, Mondays 04:30 |

The leaf covers `*.home.arpa` and `home.arpa`, with the names in a
`subjectAltName` because iOS and Chrome ignore the common name entirely.

**The leaf is valid for 397 days, and that number is not arbitrary.** Safari on
iOS and Chrome reject any server certificate issued with more than 398 days of
validity. A ten-year leaf would have looked more convenient and failed on exactly
the devices the CA was created to serve. The root is long-lived because renewing
it means revisiting every device by hand; the leaf is short-lived and renewed
automatically.

A wildcard covers a single label: `pass.home.arpa` is matched, `a.b.home.arpa` is
not. Every internal name currently in use is a single label.

Holding the signing key on the proxy is a deliberate trade. It is what allows
renewal to be automatic, and in exchange a compromise of `rpi-01` would permit
impersonating any internal name. That was already true of a node terminating TLS
for every internal service, so it adds no new class of risk.

### Naming Note

The site filenames do not match what they serve, which is an easy source of
confusion when editing them:

| File | `server_name` |
|---|---|
| `sites-available/hannah` | the DuckDNS hostname (retired) |
| `sites-available/hannah-ai` | the Cloudflare tunnel hostname (live) |

Renaming them to match is worth doing at some point.

## Current State

The node is stable and has been up for weeks at a time. Disk usage is minimal
(around 3 GB of 469 GB), and memory pressure is negligible, so there is
considerable headroom for additional workloads.

## Operational Risks

The concentration of roles on this node creates real exposure:

- **Direct internet reachability over IPv6.** Closed at the host as of
  2026-08-13. `ufw` had been allowing ports `80` and `443` from any source,
  including IPv6 — a leftover from the retired DuckDNS vhost. The full nginx log
  history contained 264 requests from 145 distinct external IPv6 addresses and
  not one legitimate external client. The inbound rule on the router is still
  open, so the node remains scannable even though it now refuses the traffic. See
  [`docs/network.md`](../network.md#retired-exposure).
- **Single point of failure for DNS.** If `rpi-01` goes down, name resolution
  depends on the secondary instance on `dns`. Failover has been tested and works.
- **No tier isolation.** The node that accepts public traffic through the
  Cloudflare tunnel is the same node that serves internal DNS and terminates the
  VPN. A compromise at the edge lands directly on an internal service host.
- **Undocumented VPN peers.** The WireGuard peer list and key rotation policy are
  not recorded anywhere. Client profiles also need `DNS` pointing at AdGuard, or
  `.home.arpa` names fail to resolve away from home even though the tunnel is up.
- **The CA root key is a single point of failure.** Losing it does not destroy
  data, but it forces creating a new authority and reinstalling it on every
  device by hand. It is included in the configuration backup for that reason.

## Intended Direction

- close the inbound IPv6 firewall rule on the router, so the node stops being
  scannable rather than merely refusing what arrives
- serve the remaining internal names over TLS with the internal certificate,
  which now costs nothing extra per name
- keep the current edge services as the node's primary role
- record the WireGuard peer inventory and confirm client profiles set `DNS`
- use the DuckDNS hostname as the WireGuard client `Endpoint` so the tunnel
  recovers on reconnect instead of requiring a manual address lookup
- evaluate whether public exposure should move off this node
- rename the nginx site files to match their `server_name`
- retain cluster and Kubernetes experimentation as a longer-term goal
