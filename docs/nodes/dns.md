# dns

## Summary

`dns` is a virtual machine on `virt` that runs a secondary AdGuard Home instance,
providing DNS redundancy for the primary resolver on `rpi-01`.

It is published internally under two names, `adguard2.home.arpa` and
`osiris.home.arpa`, both pointing at the same backend.

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

- Platform: Ubuntu Server
- Version: `24.04` (installed from `ubuntu-24.04.4-live-server-amd64.iso`)

## Services

| Service | Port | Status |
|---|---|---|
| AdGuard Home — DNS | `53` | Open |
| AdGuard Home — admin UI | `8080` | Open |
| SSH | `22` | Open |

## Documentation Status

This node was discovered during an infrastructure audit rather than being
documented when it was created. The details above were gathered from the
hypervisor configuration and external port checks.

Confirmed since:

- both AdGuard Home instances use the same filter lists
- DNS failover from `rpi-01` to this node has been tested and works

The following still needs to be verified from inside the guest:

- exact OS patch level and kernel version
- whether `node_exporter` is installed and scraped by `monitor`

## Naming Note

Three names refer to this machine: the Proxmox VM is called `dns`, while the
reverse proxy on `rpi-01` publishes it as both `adguard2.home.arpa` and
`osiris.home.arpa`. Consolidating on a single name would reduce confusion.

## Operational Notes

- failover has been tested and works, and both instances share the same filter lists
- the node runs on the same physical host as the rest of the VM fleet, so a
  failure of `virt` takes out this resolver along with everything else, leaving
  only `rpi-01` serving DNS. The reverse is well covered; this direction is not.
