# Storage Layout

## Summary

This document describes how storage is currently distributed across the homelab and how that layout is expected to evolve.

The current model is intentionally simple:

- each host keeps the minimum local storage required for its own operating system and service runtime
- centralized shared data lives on `vault`
- backups live on a dedicated logical volume on `vault`, outside the Samba shares

## Storage by Node

### virt

**Role**  
Virtualization host

**Local Storage**
- 500 GB WD Blue SSD

**Usage**
- Proxmox host installation
- VM disks stored on `local-lvm`

**Current VM Storage**
- `monitor` disk on `local-lvm` (25 GB)
- `synthia` disk on `local-lvm` (120 GB)
- `dns` disk on `local-lvm` (32 GB)

**Notes**
- VM storage is local to the host
- `vault` exports `/srv/backup/dump` over NFS to this host only, added in
  Proxmox as the `vault-dump` storage for weekly `vzdump` backups

---

### vault

**Role**  
NAS and centralized shared storage

**OS Storage**
- 1 TB WD Blue HDD

**Data Storage**
- 2 × Toshiba N300 4 TB
- RAID 1 mirror using `mdadm`
- mounted at `/srv/nas`

**Primary Data Areas**
- `/srv/nas/backups` — staging area where other nodes drop their dumps
- `/srv/nas/content`
- `/srv/nas/content/media`
- `/srv/nas/shared`
- `/srv/nas/users`
- `/srv/nas/users/drocho`

**Backup Storage**
- `lv_backup`, 400 GB on `vg0` (OS disk), mounted at `/srv/backup`
- holds the restic repository (`/srv/backup/restic`) and the `vzdump` target
  (`/srv/backup/dump`), deliberately outside the Samba shares
- a third restic repository lives on an external off-site drive (label
  `homelab-offsite`), mounted at `/srv/offsite` only while plugged in
- see [`docs/runbooks/backups.md`](../docs/runbooks/backups.md)

**Purpose**
- central media storage
- personal storage
- shared storage
- backup target, local and staging for off-site

---

### atlas

**Role**  
Jellyfin media server

**Local Storage**
- 500 GB local disk

**Current Paths**
- Media: `/mnt/media_nas`, SMB mount of `//192.168.1.21/media` on `vault`
- Config: `/srv/jellyfin/config`
- Cache: `/srv/jellyfin/cache`
- Compose files: `/srv/jellyfin/docker-compose.yml`, `/srv/vaultwarden/docker-compose.yml`
- Vaultwarden data: `/srv/vaultwarden/data`

**Notes**
- media migration to `vault` is complete; the library is consumed over SMB
- `/srv/media` still holds 126 GB of pre-migration media, including 14 files
  (29 GB) that exist nowhere else; see
  [`docs/nodes/atlas.md`](../docs/nodes/atlas.md#leftover-local-media)

---

### rpi-01

**Role**  
Edge node: DNS, VPN, reverse proxy, Cloudflare tunnel

**Local Storage**
- 512 GB NVMe

**Usage**
- AdGuard Home, nginx, WireGuard, cloudflared, dnsmasq (PXE), internal CA
- disk usage is minimal (~3 GB), leaving ample headroom for future work
- service configuration is backed up nightly into the staging area on `vault`

---

### monitor

**Role**  
Monitoring VM

**Storage**
- 25 GB virtual disk on `virt` `local-lvm`

**Usage**
- Prometheus
- Grafana

---

### synthia

**Role**  
AI workload VM

**Storage**
- 120 GB virtual disk on `virt` `local-lvm`

**Usage**
- Ollama runtime
- model-serving layer for the AI project

---

### dns

**Role**  
Secondary DNS VM

**Storage**
- 32 GB virtual disk on `virt` `local-lvm`

**Usage**
- AdGuard Home (secondary instance)

## Current Data Placement Model

Current practical model:

- service runtimes live locally on the hosts that run them
- centralized shared data belongs on `vault`
- media is fully centralized: Jellyfin on `atlas` consumes the library over SMB

## Migration State

### Already Centralized
- NAS-backed shared storage on `vault`
- Jellyfin media library, consumed over SMB from `vault`

### Remaining Cleanup
- `/srv/media` on `atlas` still holds the pre-migration copy; 14 orphaned files
  must be consolidated onto `vault` before it can be reclaimed

## Observations

The homelab currently follows a reasonable pattern:

- local storage for service runtime
- centralized storage for shared data
- clear distinction between compute nodes and storage node
- backup storage isolated from the shares that ransomware could reach

## Pending Improvements

- consolidate the orphaned files on `atlas` and reclaim `/srv/media`
- perform the tier 2 restore test (restore a VM to a scratch VMID)
- keep this inventory in sync as storage evolves