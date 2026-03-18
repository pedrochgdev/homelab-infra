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
| `virt` | `192.168.1.20` | Physical host | Proxmox virtualization host | Proxmox VE 9.1.0 | Active |
| `vault` | `192.168.1.21` | Physical host | NAS | Ubuntu Server 24.04 | Active |
| `atlas` | `192.168.1.22` | Physical host | Jellyfin media server | Ubuntu Server 24.04 | Active |
| `rpi-01` | `192.168.1.30` | Physical host | Raspberry Pi lab node | Ubuntu Server 24.04 | Active |
| `monitor` | `192.168.1.50` | VM | Monitoring | Ubuntu Server 24.04 | Active |
| `synthia` | `192.168.1.51` | VM | AI workload / Ollama | Ubuntu Server 24.04 | Active |

## Notes

- All documented homelab infrastructure currently resides on the same LAN.
- No VLAN segmentation is currently in place.
- Infrastructure services are exposed internally only.
- Future growth is expected to maintain the general range-based addressing convention unless a more structured segmented design is introduced.