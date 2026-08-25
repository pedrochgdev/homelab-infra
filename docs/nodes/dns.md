# dns

## Summary

`dns` is a virtual machine on `virt` that runs a secondary AdGuard Home instance,
providing DNS redundancy for the primary resolver on `rpi-01`.

It is published internally as `adguard2.home.arpa`. Earlier revisions also
listed `osiris.home.arpa` here, but that name actually points at Jellyfin on
`atlas`.

## Role

- secondary internal DNS resolver
- failover target if AdGuard Home on `rpi-01` becomes unavailable

## Host Platform

- Hosted on: `virt`, VMID 102
- Virtualization platform: Proxmox VE `9.1.6`

## Resources

- CPU: 1 socket, 1 core
- Memory: 2 GB RAM
- Disk: 32 GB (`local-lvm`)
- IP: `192.168.1.52`
- MAC: `BC:24:11:2B:FC:F2` on bridge `vmbr0`
- Firmware: OVMF (UEFI), machine type `q35`

## Operating System

- Platform: Ubuntu Server `24.04.4 LTS`
- Kernel: `6.8.0-124-generic`, with `6.8.0-137` installed and awaiting a reboot
- Disk usage: 5.7 GB of 31 GB

## Services

| Service | Port | Reachable from |
|---|---|---|
| AdGuard Home — DNS | `53` | LAN and WireGuard |
| AdGuard Home — admin UI | `8080` | LAN and WireGuard |
| SSH | `22` | LAN |
| `node_exporter` | `9100` | `monitor` only |

## Firewall

```
22/tcp     ALLOW  192.168.1.0/24
53/tcp,udp ALLOW  192.168.1.0/24
53/tcp,udp ALLOW  10.8.0.0/24
8080/tcp   ALLOW  192.168.1.0/24
8080/tcp   ALLOW  10.8.0.0/24
9100/tcp   ALLOW  192.168.1.50
```

AdGuard binds to all interfaces, and the node holds a globally routable IPv6
address, so the question of whether this is an open resolver is a fair one. It is
not: `ufw` has `IPV6=yes` with `DEFAULT_INPUT_POLICY="DROP"`, and every rule above
is scoped to an IPv4 range, so nothing matches over IPv6.

Verified rather than assumed — a query to port `53` on the node's global IPv6
address times out, while the same query to its IPv4 address answers normally.
This matters because an open resolver is one of the few misconfigurations that
gets a home connection used for amplification attacks.

Note that no `(v6)` rules appear in `ufw status`. That is expected here and is not
evidence of unfiltered IPv6: rules scoped to an IPv4 subnet have no IPv6
counterpart to display.

## Audit Status

This node was discovered during an infrastructure audit rather than documented
when it was created, and until 2026-08-13 everything recorded about it came from
the hypervisor configuration and external port checks. It has now been audited
from inside the guest.

Confirmed against the primary resolver, by comparing normalised dumps of both
configurations:

- identical filter lists, with the same list enabled and the same one disabled
- identical rewrites, upstreams and bootstrap servers
- identical protection, filtering, safe-search and blocking settings
- no custom rules or exceptions on either
- DNS failover from `rpi-01` has been tested and works

The claim that both instances share the same filters was correct. It had simply
never been checked against this machine.

Two genuine differences:

| | `rpi-01` | `dns` |
|---|---|---|
| AdGuard version | `v0.107.77` | `v0.107.73` |
| `ratelimit` | `0` (disabled) | `20` |

The version drift is minor but will widen, since nothing updates this node
automatically. The rate limit difference is the more interesting one: the
*primary* resolver is the one with no limit. That is harmless while the port is
firewalled and would matter immediately if it ever were not.

## Naming Note

Three names refer to this machine. The Proxmox VM is called `dns`, the guest's
own hostname is **`dns-fallback`**, and the reverse proxy on `rpi-01` publishes
it as `adguard2.home.arpa`. A fourth name, `osiris.home.arpa`, was long recorded
as an alias of this machine; the running proxy shows it points at Jellyfin.

The hostname was only discovered once the node was reachable over SSH; every
earlier document assumed it matched the VM name. Consolidating on one name would
reduce confusion.

## Operational Notes

- failover has been tested and works, and both instances are confirmed equivalent
- the node runs on the same physical host as the rest of the VM fleet, so a
  failure of `virt` takes out this resolver along with everything else, leaving
  only `rpi-01` serving DNS. The reverse is well covered; this direction is not.
- nothing updates AdGuard here, so it drifts behind the primary over time
- a reboot is pending: the running kernel is 13 versions behind the installed one
