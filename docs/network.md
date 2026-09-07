# Network

## Summary

The homelab currently operates on a single flat LAN with static addressing for infrastructure nodes and a separate DHCP pool for non-lab devices. Network segmentation has not yet been introduced, but future work is expected to include VLAN design and more formal network separation.

Most services are internal-only, but the network is no longer strictly internal:
`rpi-01` acts as an edge node providing internal DNS, the Tailscale VPN entry
(a subnet router for the LAN), a reverse proxy, and a Cloudflare tunnel that
publishes one project externally. See
[Access Model](#access-model) for the full picture.

## Primary Network

- Network: `192.168.1.0/24`
- Gateway / Router: `192.168.1.1`
- DNS: AdGuard Home on `rpi-01`, with a secondary instance on `dns` (`192.168.1.52`)
- DHCP Pool: `192.168.1.100` to `192.168.1.199`
- Addressing for homelab nodes: static manual IP assignment
- IPv6: globally routable addresses in `2001:1388:80d:f6c6::/64`, delegated by the ISP router and currently unmanaged. **The delegated prefix changes**: it was `…:f3f3::/64` when first documented and every node had moved to `…:f6c6::/64` by 2026-08-13. Anything keyed to a specific IPv6 address — a firewall rule, a DNS record, a document — goes stale silently when that happens

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
| `osiris.home.arpa` | `http://192.168.1.22:8096` (Jellyfin) |
| `pass.home.arpa` | `https://192.168.1.22:8222` (Vaultwarden, over TLS) |
| `ca.home.arpa` | Static: the internal CA root certificate |
| `tarjeta.home.arpa` | `http://192.168.1.22:8090` (Tarjetero, over TLS) |
| `rutina.home.arpa` | `http://192.168.1.22:8091` (Rutina, over TLS) |

`osiris.home.arpa` is the Jellyfin vhost — "Osiris" is the Jellyfin server's
own configured name. Earlier revisions recorded it as a second alias of the
AdGuard instance on `dns`; checking the running proxy shows that was never
true.

All `.home.arpa` names resolve through a single wildcard rewrite in AdGuard,
`*.home.arpa → 192.168.1.30`, present on both instances. Adding a name therefore
requires an nginx vhost and no DNS change at all.

`pass.home.arpa`, `tarjeta.home.arpa` and `rutina.home.arpa` are served over
`443` with a certificate signed by the internal CA on `rpi-01`, with `80`
redirecting to `443`. The other names are still plain `80`; extending the certificate
to them costs nothing further and is listed under planned work.

Note for any future vhost on this node: nginx here is `1.24.0`, which predates
the `http2 on;` directive introduced in `1.25.1`. Using the modern form fails
the config test with `unknown directive` and leaves the configuration unable to
reload. Declare it the old way, `listen 443 ssl http2;`.

## Remote Access

There are currently three distinct remote access paths:

**Tailscale.** Since 2026-08-29 the VPN is Tailscale. `tailscaled` on `rpi-01`
acts as a subnet router advertising `192.168.1.0/24`, so any device in the
tailnet reaches the entire LAN, not just the node. Because every connection is
outbound (WireGuard-based NAT traversal through Tailscale's coordination
servers), it works from IPv4-only networks and behind the ISP's carrier-grade
NAT — exactly the limitation that made plain WireGuard unusable away from
IPv6-capable networks. The first client, a phone, established a direct
connection with no DERP relay in the path.

Node-side configuration worth recording: `--advertise-routes=192.168.1.0/24`
(the subnet route must also be approved in the admin console),
`--accept-dns=false` so the node keeps its own resolver, and
`--operator=drocho` so status can be queried without root. `ufw` additionally
allows traffic in on `tailscale0` and forwarding from `tailscale0` to `eth0`.

`.home.arpa` names do not resolve for tailnet clients by themselves — their DNS
never touches AdGuard. The fix is split DNS in the Tailscale admin panel:
domain `home.arpa` → nameserver `192.168.1.30`. Without it, clients reach
services by IP only.

**WireGuard — disabled 2026-08-29.** Until then `rpi-01` terminated a WireGuard
tunnel on `wg0` (`10.8.0.0/24`, node at `10.8.0.1`). It was replaced because it
was reachable **over IPv6 only, and that was not a choice**: the connection is
behind carrier-grade NAT — a traceroute leaves the router into three
consecutive hops in `10.0.0.0/8` — so no inbound IPv4 path exists or can be
created. A VPN that cannot be used from a network without IPv6 (most university
and corporate WiFi) kept failing exactly when it was needed. The service is
disabled, not removed: configuration and peer keys remain in `/etc/wireguard/`
and in the nightly configuration backup, so reverting is a `systemctl enable
--now wg-quick@wg0` plus the router rule. The router's inbound IPv6 allowance
for `UDP 51820` no longer serves anything and should be removed.

The ISP rotates the delegated prefix roughly monthly, sometimes twice within
days:

```
2026-05-03  2001:1388:80d:fc3::/64      2026-07-02  …:f3f3::/64
2026-06-02  …:fde8::/64                 2026-08-02  …:c2a3::/64
2026-06-05  …:169f::/64                 2026-08-05  …:f6c6::/64
```

Only the prefix changes; the interface identifier derives from the MAC and is
stable. Address lifetimes are about 3 days valid and 2 days preferred, so for
roughly 24 hours after a rotation the old address is still present but deprecated
— which is why anything selecting an address must exclude deprecated and
temporary ones rather than taking the first it finds.

**Cloudflare tunnel.** `cloudflared` runs on `rpi-01` and is the only intended
public entry point. It establishes an outbound connection to Cloudflare and
publishes a single external hostname to the local nginx instance on
`localhost:80`. That vhost proxies to a personal project running on the
workstation (`192.168.1.109`), not to homelab infrastructure. TLS is terminated
by Cloudflare, so no certificate is managed locally for this path.

**Dynamic DNS.** `rpi-01` publishes its own global IPv6 address as an `AAAA`
record; it existed so the WireGuard `Endpoint` on client devices was a name
rather than an address that expires. With WireGuard disabled the record serves
no active purpose — Tailscale needs no inbound address at all — but the updater
is harmless and is left running in case WireGuard returns. Earlier revisions of this document said the updater "lives
elsewhere, most likely the router". It does not, and never did: it has always run
on `rpi-01`.

The record is `vpn.<zone>` in a Cloudflare zone, TTL 60, **deliberately not
proxied** — Cloudflare's proxy does not forward UDP and would break WireGuard
without any obvious symptom.

This replaced a DuckDNS updater that had three defects worth recording, since
they are easy to repeat:

- the API token sat in a world-readable script, so any local account could
  repoint the record
- `curl -sk` disabled certificate validation while transmitting that token
- **it recorded every attempt as successful.** If the provider had returned an
  error, the history file would still say the address was published, and nothing
  would retry until the next rotation

The replacement compares against what the DNS provider actually holds rather than
against a local state file. A state file records what was believed to be
published; only the API knows what is.

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

### The host side was still open

The remediation above was incomplete, and a later audit found why. `ufw` on
`rpi-01` had been accepting ports `80` and `443` **from any source, IPv4 and
IPv6 alike** — four rules left behind when the DuckDNS vhost was withdrawn. The
vhost was removed; the door it needed was not.

The full retained nginx log history contained **264 requests from 145 distinct
external IPv6 addresses**, and not a single legitimate external client. The
payloads were a protocol sweep rather than web traffic: OPC UA, the Bitcoin wire
protocol, DICOM, iSCSI, Oracle TNS, LDAP StartTLS, XMPP. All 564 requests for the
published project arrived over loopback from `cloudflared`, never through the
open port.

This mattered more than the catch-all suggested, because **the catch-all only
stops hostnames nginx does not recognise**. Verified against the running proxy:

```
Host: proxmox.home.arpa   ->  HTTP 200   (Proxmox interface)
Host: monitor.home.arpa   ->  HTTP 302   (Grafana)
Host: osiris.home.arpa    ->  HTTP 302   (Jellyfin)
Host: unknown.example.com ->  444        (catch-all)
```

Any external client sending a `Host` header nginx knows was proxied into the LAN.
None of those names are secret, and `proxmox.home.arpa` is close to the first
name anyone would try.

Remediation, 2026-08-13: the four unscoped rules were deleted, leaving `80` and
`443` reachable only from `192.168.1.0/24` and `10.8.0.0/24`. The Cloudflare
tunnel is unaffected because `cloudflared` delivers over loopback, which `ufw`
does not filter. Verified after the change: the tunnel still serves and the
internal proxy still answers.

Still outstanding: whatever on the router still permits inbound traffic to
`rpi-01`. The host refuses it on its own now, so the exposure is closed, but until
the router stops accepting it the node stays reachable and therefore scannable.

The router configuration has not been inspected, so what follows is inference
from evidence rather than a record of what is there.

**The way in was almost certainly IPv6, not IPv4.** The connection has a real
routable public IPv4 address, not CGNAT. Port `80` on such an address is scanned
many times a day, yet three weeks of nginx logs contain **zero** IPv4 requests
against thousands over IPv6. Had `80` or `443` been forwarded on IPv4, that
traffic would be unmistakable.

This means two different places in the router need checking, and only one of them
is "port forwarding":

**IPv4 — port forwarding.** NAT means an explicit rule is needed for anything to
reach a host inside. Expect to find `51820/udp → 192.168.1.30` here. That rule
carried WireGuard; since the move to Tailscale (outbound-only) on 2026-08-29 it
protects nothing and **can be removed along with the rest**. Remove `80` or
`443` entries if any exist.

**IPv6 — firewall.** There is no NAT in IPv6; every node already holds its own
globally routable address, so nothing is being "forwarded". A router firewall
decides whether inbound traffic is allowed, and consumer routers frequently ship
with it disabled or permissive. In that case there is no specific rule to delete
— there is a setting to enable.

That explanation fits the evidence: scanners reached `rpi-01` because it was the
only node with a service listening *and* permitted by its own firewall. The same
packets very likely reached every other node too, and were dropped by `ufw`
without anyone noticing.

### Confirmed closed

The `80` and `443` entries were removed from the router on 2026-08-13, leaving the
`51820/udp` rule that carries WireGuard. That rule turned out to be an **IPv6**
firewall entry, not an IPv4 port forward — no IPv4 forwarding rules were ever
created, which fits the connection being behind carrier-grade NAT and explains
the complete absence of IPv4 scanner traffic.

This is worth stating plainly because the obvious inference was wrong: the
assumption that WireGuard must depend on an IPv4 forward would, if acted on, have
closed the IPv6 side and taken remote access to the whole homelab with it.

Verification needed the right instrument. The nginx log cannot answer the
question any more, because `ufw` now drops these packets before nginx sees them —
the log would look identical whether the router was fixed or not. The kernel's
`UFW BLOCK` records can, because they show what still arrives:

```
blocks in 6 hours, by source and destination port
    5  192.168.1.123  TCP 853
    5  192.168.1.104  TCP 853
    5  10.8.0.3       TCP 853

blocks from a global IPv6 source, 24 hours:  0
```

Every packet the firewall had to drop originated inside the LAN or the VPN. None
came from the internet, where previously they arrived around a dozen times a day.
Nothing is reaching the host at all, which is what closing the router rule was
supposed to achieve.

### Devices asking for encrypted DNS

The same measurement turned up something unrelated: three devices repeatedly
attempt DNS-over-TLS on port `853` against AdGuard, and are refused because DoT
is not configured. This is Android's "Private DNS" in automatic mode.

Nothing breaks — the clients fall back to plain DNS on `53` — but they retry
indefinitely and DNS travels the LAN in the clear. Now that an internal CA issues
for `*.home.arpa`, serving DoT costs little: AdGuard can present the existing
certificate on `853`.

One trap to avoid if this is done: setting Android's Private DNS to a *hostname*
under `.home.arpa` breaks DNS entirely whenever the device is away from home
without the VPN, because the name does not resolve out there. Leaving it on
automatic gets encrypted DNS at home and a working phone everywhere else.

The general lesson is worth keeping: **host-based routing is not access control.
A reverse proxy is a set of doors into the network, and what decides who may
knock is the firewall.**

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

### `rpi-01`

```
22/tcp           ALLOW  192.168.1.0/24
22               ALLOW  10.8.0.0/24
53/tcp,udp       ALLOW  192.168.1.0/24
53/tcp,udp       ALLOW  10.8.0.0/24
80/tcp           ALLOW  192.168.1.0/24
80/tcp           ALLOW  10.8.0.0/24
443/tcp          ALLOW  192.168.1.0/24
443/tcp          ALLOW  10.8.0.0/24
51820/udp        ALLOW  Anywhere            (WireGuard; removable since the move to Tailscale)
9100/tcp         ALLOW  192.168.1.50
69/udp           ALLOW  192.168.1.0/24      (TFTP)
4011/udp         ALLOW  192.168.1.0/24      (PXE)
10000:10100/udp  ALLOW  192.168.1.0/24      (TFTP data)
67/udp on eth0   ALLOW  Anywhere            (PXE DHCP, source is 0.0.0.0)
```

`51820/udp` was open to the world by necessity while WireGuard was the VPN
entry point; with WireGuard disabled it answers nothing and can be removed,
together with the `10.8.0.0/24` allowances above. Tailscale added two rules not
yet shown in this listing: `allow in on tailscale0` and a route allowance from
`tailscale0` to `eth0`. The two `Anywhere` rules that did not belong were
removed on 2026-08-13; see [Retired Exposure](#retired-exposure).

The `69`, `4011`, `10000:10100` and `67` rules belong to the remote boot service,
documented in [`docs/services/remote-boot.md`](services/remote-boot.md).

Note there is no rule for `8080`, where AdGuard's admin interface listens. It is
reachable only through the reverse proxy, which reaches it over loopback. That is
the stricter arrangement, and it is worth not "fixing".

### `dns`

```
22/tcp     ALLOW  192.168.1.0/24
53/tcp,udp ALLOW  192.168.1.0/24
53/tcp,udp ALLOW  10.8.0.0/24
8080/tcp   ALLOW  192.168.1.0/24
8080/tcp   ALLOW  10.8.0.0/24
9100/tcp   ALLOW  192.168.1.50
```

Both resolvers bind AdGuard to all interfaces and both hold globally routable
IPv6 addresses, which raises the obvious question of whether either is an open
resolver. Neither is: `ufw` on both has `IPV6=yes` and `DEFAULT_INPUT_POLICY`
`DROP`, and every rule is scoped to an IPv4 range, so nothing matches over IPv6.

Confirmed by measurement — a DNS query to a node's global IPv6 address on port
`53` times out, while the same query to its IPv4 address answers. The absence of
`(v6)` lines in `ufw status` is not evidence of unfiltered IPv6; rules scoped to
an IPv4 subnet simply have no IPv6 counterpart to print.

Undocumented here: `atlas`, `monitor` and `synthia`, whose rule sets have not
been recorded yet.

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
- `ufw` rule sets are documented for `vault` and `rpi-01`; `atlas`, `monitor` and `synthia` are still unrecorded
- IPv6 is delegated by the ISP and globally routable, but not planned, documented, or firewalled deliberately; an inbound allowance to `rpi-01` went unnoticed through two separate audits before the host-side rules were found
- services bound to `::` have not been audited against the router's IPv6 policy; `vault` listens on `[::]:445` for Samba and `virt` exposes the Proxmox interface on `*:8006`
- public exposure terminates on the same node that serves internal DNS and VPN, so the edge tier has no isolation from the internal tier
- the reverse proxy depends on a workstation address inside the DHCP pool
- remote access now depends on Tailscale's coordination service, a third party; the outage mode is losing new connections, since established tunnels are peer-to-peer
- the WireGuard leftovers (router rule for `51820`, `ufw` allowances for `10.8.0.0/24`) are still in place though nothing uses them

## Planned Network Improvements

- remove the inbound IPv6 rule on the router, the last piece of the retired exposure
- introduce VLANs for more structured separation, starting by isolating the edge tier
- define an IPv6 posture: either disable it or plan and firewall it deliberately
- serve the remaining internal hostnames over TLS using the internal CA
- record the `ufw` rules of `atlas`, `monitor` and `synthia`
- give the workstation a static address or DHCP reservation
- remove the WireGuard leftovers: the router's `51820/udp` IPv6 rule and the `10.8.0.0/24` `ufw` allowances
- create a more professional network layout
- formalize service exposure and access boundaries
- expand network documentation as topology becomes more structured