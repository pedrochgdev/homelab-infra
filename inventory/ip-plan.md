# IP Plan

## Summary

This document tracks the current IP allocation scheme used by the homelab. The environment uses a single `192.168.1.0/24` network with static addressing for infrastructure nodes and a separate DHCP pool for general devices.

## Network

- Network: `192.168.1.0/24`
- Gateway: `192.168.1.1`
- DHCP pool: `192.168.1.100` to `192.168.1.199`
- Infrastructure addressing: static manual IP assignment

## Addressing Conventions

Current allocation ranges:

- `.10` — switch management
- `.20` to `.29` — physical servers
- `.30` onward — Raspberry Pi and future cluster-related expansion
- `.50` onward — virtual machines

## Infrastructure IP Inventory

| Hostname | IP Address | Type | Role | Platform | Status |
|---|---|---|---|---|---|
| `switch` | `192.168.1.10` | Network device | Managed switch | TP-Link SG3428 | Active |
| `virt` | `192.168.1.20` | Physical host | Proxmox virtualization host | Proxmox VE 9.1.6 | Active |
| `vault` | `192.168.1.21` | Physical host | NAS | Ubuntu Server 24.04.4 | Active |
| `atlas` | `192.168.1.22` | Physical host | Jellyfin media server | Ubuntu Server 24.04.4 | Active |
| `rpi-01` | `192.168.1.30` | Physical host | Edge node: DNS, VPN, reverse proxy | Ubuntu Server 24.04.4 | Active |
| `monitor` | `192.168.1.50` | VM | Monitoring | Ubuntu Server 24.04.4 | Active |
| `synthia` | `192.168.1.51` | VM | AI workload / Ollama | Ubuntu Server 24.04.4 | Active |
| `dns` | `192.168.1.52` | VM | Secondary DNS (AdGuard Home) | Ubuntu Server 24.04 | Active |

## Other Addresses in Use

These addresses are not infrastructure nodes but are referenced by infrastructure
configuration and are documented here so the dependency is visible:

| IP Address | Description | Referenced by |
|---|---|---|
| `192.168.1.109` | Personal workstation (DHCP range) | Prometheus scrape targets; `rpi-01` reverse proxy backends |

## Secondary Networks

| Range | Purpose | Terminates on |
|---|---|---|
| `10.8.0.0/24` | WireGuard VPN tunnel | `rpi-01` (`wg0`, gateway `10.8.0.1`) |
| `172.17.0.0/16`, `172.18.0.0/16`, `172.30.0.0/16` | Docker bridge networks | `atlas` |

## IPv6

Every node also holds a globally routable IPv6 address, currently in the
`2001:1388:80d:f6c6::/64` prefix delegated by the ISP router. IPv6 is unplanned
and undocumented at the addressing level: there is no static assignment scheme,
and addresses derive from the interface MAC.

**The delegated prefix is not stable.** It was `2001:1388:80d:f3f3::/64` when
first recorded and every node had moved to `…:f6c6::/64` by 2026-08-13, without
anything being changed locally. Any rule, record or document pinned to a specific
IPv6 address stops matching when the ISP rotates the delegation, and does so
silently — the address simply stops existing.

That instability cuts both ways. It makes IPv6 addresses useless as stable
identifiers, and it also means an inbound firewall rule written against one
address quietly stops protecting anything.

This is not theoretical. An audit found internet scanner traffic reaching nginx
on `rpi-01` over IPv6, on a path that proxied through to the personal
workstation. Details and remediation are in
[`docs/network.md`](../docs/network.md#retired-exposure).

Because IPv6 addresses are globally routable rather than NAT-hidden, any service
bound to `::` is reachable from outside the LAN if the router firewall permits
it. Services currently bound to `::` that have not been audited against the
router's IPv6 policy include Samba on `vault` (`[::]:445`, `[::]:139`), SSH on
every node, and the Proxmox interface on `virt` (`*:8006`).

Binding to all interfaces is not itself the problem, as the DNS nodes show. Both
resolvers bind AdGuard to every interface, yet neither answers over IPv6 because
`ufw` runs with `IPV6=yes` and a default `DROP` policy, and every allow rule is
scoped to an IPv4 range. Verified by querying a node's global IPv6 address on
port `53`: it times out, while IPv4 answers.

What matters is whether a host has a firewall whose default denies, not what the
service binds to. `virt` is the gap here, since the Proxmox firewall is enabled
but applied to no guest.

## Notes

- All documented homelab infrastructure currently resides on the same LAN.
- No VLAN segmentation is currently in place.
- The `.52` node falls inside the documented VM range and follows the convention.
- The workstation at `.109` sits inside the DHCP pool while infrastructure
  configuration depends on its address. A static assignment or reservation would
  make that dependency safer.
- Future growth is expected to maintain the general range-based addressing convention unless a more structured segmented design is introduced.