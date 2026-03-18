# Storage Layout

## Summary

This document describes how storage is currently distributed across the homelab and how that layout is expected to evolve.

The current model is intentionally simple:

- each host keeps the minimum local storage required for its own operating system and service runtime
- centralized shared data lives on `vault`
- some workloads still use local storage and are planned for migration later

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
- `monitor` disk on `local-lvm`
- `synthia` disk on `local-lvm`

**Notes**
- VM storage is currently local to the host
- no secondary storage tier is currently documented for Proxmox workloads

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
- `/srv/nas/backups`
- `/srv/nas/content`
- `/srv/nas/content/media`
- `/srv/nas/shared`
- `/srv/nas/users`
- `/srv/nas/users/drocho`

**Purpose**
- central media storage
- personal storage
- shared storage
- future backup target

---

### atlas

**Role**  
Jellyfin media server

**Local Storage**
- 500 GB local disk

**Current Paths**
- Media: `/srv/media`
- Config: `/srv/jellyfin/config`
- Cache: `/srv/jellyfin/cache`
- Compose file: `/srv/jellyfin/docker-compose.yml`

**Notes**
- media is still stored locally
- media is planned to be migrated to `vault`
- current likely transport is SMB, subject to future review

---

### rpi-01

**Role**  
Lab node

**Local Storage**
- 512 GB NVMe

**Usage**
- currently reserved for future experimentation
- no defined active workload at present

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

## Current Data Placement Model

Current practical model:

- service runtimes live locally on the hosts that run them
- centralized shared data belongs on `vault`
- media is not fully centralized yet because Jellyfin still depends on local storage on `atlas`

## Migration State

### Already Centralized
- NAS-backed shared storage on `vault`

### Not Yet Centralized
- Jellyfin media library on `atlas`

### Planned Change
- move media storage from `atlas` to `vault`
- keep Jellyfin service hosting on `atlas` unless a later redesign justifies moving it

## Observations

The homelab currently follows a reasonable early-stage pattern:

- local storage for service runtime
- centralized storage for shared data
- clear distinction between compute nodes and storage node

The main remaining storage improvement is to finish separating the Jellyfin application host from the media storage backend.

## Pending Improvements

- finalize media migration from `atlas` to `vault`
- define backup destination strategy
- define which directories require backup versus which can be recreated
- document retention and recovery expectations by data type