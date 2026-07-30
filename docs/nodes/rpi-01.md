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
| nginx | Internal reverse proxy and public vhosts | `80`, `443` |
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

- `00-default-deny` — explicit catch-all returning `444` for any request whose
  `Host` matches no other vhost
- `homelab` — internal `.home.arpa` hostnames mapped to Proxmox, Grafana, the
  display page, both AdGuard instances, and Jellyfin
- `hannah-ai` — the personal project vhost on port `80`, served through the
  Cloudflare tunnel and proxied to the workstation at `192.168.1.109`

Retained in `sites-available` but no longer enabled:

- `hannah` — the retired DuckDNS vhost that previously served the same project
  directly on `443` with a Let's Encrypt certificate. See
  [`docs/network.md`](../network.md#retired-exposure) for why it was withdrawn.

The full internal hostname table lives in
[`docs/network.md`](../network.md#internal-reverse-proxy). External hostnames are
deliberately not recorded in this repository.

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

- **Direct internet reachability over IPv6.** An audit found unsolicited scanner
  traffic reaching nginx on this node from external IPv6 addresses, independent
  of the Cloudflare tunnel. The serving side has been closed, but the inbound
  firewall rule on the router is still open. See
  [`docs/network.md`](../network.md#retired-exposure).
- **Single point of failure for DNS.** If `rpi-01` goes down, name resolution
  depends on the secondary instance on `dns`. Failover has been tested and works.
- **No tier isolation.** The node that accepts public traffic through the
  Cloudflare tunnel is the same node that serves internal DNS and terminates the
  VPN. A compromise at the edge lands directly on an internal service host.
- **Undocumented VPN peers.** The WireGuard peer list and key rotation policy are
  not recorded anywhere.
- **No backup of configuration.** AdGuard, nginx, WireGuard, and cloudflared
  configuration exist only on this node.

## Intended Direction

- close the inbound IPv6 firewall rule on the router
- keep the current edge services as the node's primary role
- document and back up the service configuration
- use the DuckDNS hostname as the WireGuard client `Endpoint` so the tunnel
  recovers on reconnect instead of requiring a manual address lookup
- evaluate whether public exposure should move off this node
- rename the nginx site files to match their `server_name`
- retain cluster and Kubernetes experimentation as a longer-term goal
