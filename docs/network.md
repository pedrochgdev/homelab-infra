# Network

## Summary

The homelab currently operates on a single flat LAN with static addressing for infrastructure nodes and a separate DHCP pool for non-lab devices. Network segmentation has not yet been introduced, but future work is expected to include VLAN design and more formal network separation.

Most services are internal-only, but the network is no longer strictly internal:
`rpi-01` acts as an edge node providing internal DNS, a WireGuard VPN endpoint, a
reverse proxy, and a Cloudflare tunnel that publishes one project externally. See
[Access Model](#access-model) for the full picture.

## Primary Network

- Network: `192.168.1.0/24`
- Gateway / Router: `192.168.1.1`
- DNS: AdGuard Home on `rpi-01`, with a secondary instance on `dns` (`192.168.1.52`)
- DHCP Pool: `192.168.1.100` to `192.168.1.199`
- Addressing for homelab nodes: static manual IP assignment
- IPv6: globally routable addresses in `2001:1388:80d:f3f3::/64`, delegated by the ISP router and currently unmanaged

## Addressing Scheme

The current addressing plan follows simple functional ranges:

- `192.168.1.10` — network switch management
- `192.168.1.20` to `192.168.1.29` — servers
- `192.168.1.30` onward — Raspberry Pi and future cluster-related expansion
- `192.168.1.50` onward — virtual machines

This convention is intentionally simple and is designed to remain understandable while the environment grows.

## Infrastructure IP Plan

| Hostname | IP Address | Role |
|---|---|---|
| `switch` | `192.168.1.10` | Managed switch |
| `virt` | `192.168.1.20` | Proxmox host |
| `vault` | `192.168.1.21` | NAS |
| `atlas` | `192.168.1.22` | Jellyfin media server |
| `rpi-01` | `192.168.1.30` | Edge node: DNS, VPN, reverse proxy |
| `monitor` | `192.168.1.50` | Monitoring VM |
| `synthia` | `192.168.1.51` | AI workload VM |
| `dns` | `192.168.1.52` | Secondary DNS VM |

## Physical Network Infrastructure

### Switch

- Model: TP-Link SG3428
- Management IP: `192.168.1.10`
- Role: central switching for the homelab rack and connected infrastructure

The switch is part of the documented infrastructure even though the network is still relatively simple. It will become more important as the environment grows, especially once VLANs or more structured segmentation are introduced.

## Current Topology

The physical topology is still flat, but the logical topology now has an edge tier:

- all nodes are connected to a single LAN
- no VLANs are configured
- `rpi-01` acts as the edge node for DNS, VPN, and reverse proxying
- most services are reachable only from the LAN or the VPN
- one project is published to the internet through a Cloudflare tunnel

## Internal DNS

DNS is served by AdGuard Home rather than by the router:

- primary: AdGuard Home on `rpi-01` (`192.168.1.30:53`), admin UI on port `8080`
- secondary: AdGuard Home on `dns` (`192.168.1.52:53`), admin UI on port `8080`

A private `.home.arpa` namespace is resolved internally and fronted by the
reverse proxy on `rpi-01`.

## Internal Reverse Proxy

`rpi-01` runs nginx and maps internal hostnames to service backends:

| Internal hostname | Backend |
|---|---|
| `proxmox.home.arpa` | `https://192.168.1.20:8006` |
| `monitor.home.arpa` | `http://192.168.1.50:3000` (Grafana) |
| `display.home.arpa` | `http://192.168.1.50:80` (static display page) |
| `adguard.home.arpa` | `http://127.0.0.1:8080` (AdGuard on `rpi-01`) |
| `adguard2.home.arpa` | `http://192.168.1.52:8080` (AdGuard on `dns`) |
| `osiris.home.arpa` | `http://192.168.1.52:8080` |

Jellyfin on `192.168.1.22:8096` is also proxied through this node.

## Remote Access

There are currently three distinct remote access paths:

**WireGuard.** `rpi-01` terminates a WireGuard tunnel on `wg0`, subnet
`10.8.0.0/24`, with the node itself at `10.8.0.1`. This is a direct VPN entry
point into the homelab LAN.

**Tailscale.** The personal workstation remains reachable through Tailscale, and
remote desktop into it is still usable as an administrative path to the LAN.

**Cloudflare tunnel.** `cloudflared` runs on `rpi-01` and is the only intended
public entry point. It establishes an outbound connection to Cloudflare and
publishes a single external hostname to the local nginx instance on
`localhost:80`. That vhost proxies to a personal project running on the
workstation (`192.168.1.109`), not to homelab infrastructure. TLS is terminated
by Cloudflare, so no certificate is managed locally for this path.

**Dynamic DNS.** A DuckDNS record tracks the ISP-assigned public IP. It does not
publish any service. Its purpose is recovery: when the ISP changes the public IP,
the WireGuard tunnel breaks, and the DuckDNS record is where the current address
can be looked up in order to reconnect.

A previous setup also served the project directly over `443` on the DuckDNS
hostname, using a Let's Encrypt certificate from Certbot. That vhost has been
retired; see [Retired Exposure](#retired-exposure) below.

External hostnames are deliberately omitted from this repository.

## Service Exposure

Internal only, LAN or VPN:

- SMB shares on `vault`
- Jellyfin on `atlas`
- Prometheus and Grafana on `monitor`
- Ollama service on `synthia`
- Proxmox management on `virt`
- AdGuard Home admin interfaces on `rpi-01` and `dns`

Reachable from the internet:

- one personal project vhost, published through the Cloudflare tunnel and
  proxied by `rpi-01` to the workstation

## Retired Exposure

An infrastructure audit found that `rpi-01` was directly reachable from the
internet over IPv6 on ports `80` and `443`, independently of the Cloudflare
tunnel. The nginx access logs contained unsolicited scanner traffic from Vultr,
DigitalOcean, and Linode IPv6 ranges, probing for known appliance endpoints
(Fortinet `/remote/login`, Pulse Secure `/dana-na/`, SonicWall, `/wsman`) and
identifying as LEAKIX.

Some of those requests returned `200`, and others returned `502` — meaning nginx
accepted the external request and attempted to proxy it to the workstation at
`192.168.1.109:3000`, failing only because the backend was down. The path from
the internet through to the workstation was live.

The exposure was scoped, not general: across the audited nodes there were zero
failed SSH password attempts (31,273 lines of `auth.log` on `rpi-01`, 11,832 on
`vault`), so inbound IPv6 was not open to arbitrary ports. The likely cause was a
firewall allowance created to support Certbot HTTP-01 validation for the retired
`443` vhost.

Remediation applied:

- the DuckDNS vhost was removed from `sites-enabled` (the definition is retained
  in `sites-available` so the change is reversible), closing port `443`
- an explicit catch-all `default_server` was added that returns `444` for any
  request whose `Host` does not match a defined vhost
- the Let's Encrypt certificate for the DuckDNS hostname was deleted, so nothing
  is renewed for a name that no longer serves anything

Verified after the change: unknown `Host` values and bare-IP requests are cut
with `444`, while every internal `.home.arpa` name and the Cloudflare tunnel
vhost continue to route correctly.

The catch-all matters more than it appears. nginx falls back to the
first-loaded vhost when no `default_server` is declared, so removing the DuckDNS
vhost alone would have promoted the project vhost to implicit default and started
proxying unmatched scanner traffic straight to the workstation.

Still outstanding: the inbound IPv6 firewall rule on the router should be
removed. Until then the host remains reachable and scannable, even though nginx
now refuses to serve those requests.

## Host Firewalls

Earlier revisions of this document stated that no filtering existed on the
nodes. That was wrong: every Ubuntu node runs `ufw`, and `virt` runs the Proxmox
firewall.

| Node | Firewall | State |
|---|---|---|
| `vault` | `ufw` | Active, rules documented below |
| `atlas` | `ufw` | Active |
| `rpi-01` | `ufw` | Active |
| `monitor` | `ufw` | Active |
| `synthia` | `ufw` | Active |
| `virt` | Proxmox firewall | Enabled, but see caveat |

### `vault`

```
22/tcp     ALLOW  192.168.1.0/24
Samba      ALLOW  192.168.1.0/24
9100/tcp   ALLOW  192.168.1.50
9633/tcp   ALLOW  192.168.1.50
111,2049   ALLOW  192.168.1.20        (NFS, added for VM backups)
```

These are well scoped. The metrics exporters accept connections only from
`monitor` rather than from the whole LAN, and the NFS export is reachable only
from `virt`.

### `virt` and its guests

The Proxmox firewall is enabled at cluster level with `policy_in: DROP`. There
is no `host.fw`, and more importantly **no VM has `firewall=1` set on its
network device**, so the drop policy is not applied to any guest.

In practice this means `monitor`, `synthia` and `dns` are unfiltered at the
hypervisor level and rely entirely on their own `ufw`. The Proxmox firewall
being "enabled" is misleading if read without that detail.

The rule sets on the nodes other than `vault` have not been recorded here yet.

## Current Limitations

- no network segmentation
- no VLAN isolation
- no dedicated management network
- the Proxmox firewall is enabled but not applied to any guest, since no VM has `firewall=1` on its network device
- `ufw` rule sets are documented only for `vault`
- IPv6 is delegated by the ISP and globally routable, but not planned, documented, or firewalled deliberately; an inbound allowance to `rpi-01` went unnoticed until an audit found scanner traffic in the logs
- services bound to `::` have not been audited against the router's IPv6 policy; `vault` listens on `[::]:445` for Samba and `virt` exposes the Proxmox interface on `*:8006`
- public exposure terminates on the same node that serves internal DNS and VPN, so the edge tier has no isolation from the internal tier
- the reverse proxy depends on a workstation address inside the DHCP pool
- WireGuard peer inventory is not documented
- nothing on `rpi-01` refreshes the DuckDNS record, so the updater lives elsewhere (most likely the router) and has not been verified

## Planned Network Improvements

- introduce VLANs for more structured separation, starting by isolating the edge tier
- define an IPv6 posture: either disable it or plan and firewall it deliberately
- give the workstation a static address or DHCP reservation
- document WireGuard peers and key rotation
- create a more professional network layout
- formalize service exposure and access boundaries
- expand network documentation as topology becomes more structured