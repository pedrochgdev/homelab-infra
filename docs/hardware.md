# Hardware

## Summary

This document tracks the current hardware assigned to the homelab and its intended role within the environment. It is intended to provide a concise operational inventory rather than exhaustive benchmark-level detail.

## Node Inventory

### virt

**Role**  
Primary virtualization host

**Operating System**  
Proxmox VE 9.1.6 (Debian 13 "trixie", kernel `6.17.13-2-pve`)

**Purpose**  
Run virtual machines and supporting infrastructure services

**Hardware**
- CPU: Intel Core i7-9700K (8 cores)
- Memory: 16 GB DDR4 RAM
- GPU: NVIDIA GeForce RTX 2060, passed through to `synthia`
- Integrated graphics: Intel UHD Graphics 630, used by the host
- Storage: 500 GB SSD (`sda`, ~466 GB), LVM-thin `local-lvm` for VM disks

**Current Virtual Machines**
- VMID 100 `ia`: AI workload VM, guest hostname `synthia`, with RTX 2060 passthrough
- VMID 101 `monitor`: monitoring VM running Prometheus and Grafana
- VMID 102 `dns`: secondary AdGuard Home instance at `192.168.1.52`

**Notes**
- Main compute node for virtualization
- The Proxmox VM name for the AI workload is `ia`, while the guest hostname is `synthia`. Both names appear across the documentation and tooling.
- Current AI workload behavior suggests VRAM is a more significant constraint than system RAM for the existing Ollama deployment
- Host memory is fairly committed: roughly 10 GB of 16 GB in use with three VMs running

---

### vault

**Role**  
Dedicated NAS server

**Operating System**  
Ubuntu Server 24.04

**Purpose**  
Provide centralized network storage for media and future shared data

**Hardware**
- CPU: Intel Core i7-3770 (8 threads)
- Memory: 8 GB DDR3 RAM
- GPU: None
- OS Storage: 1 TB WD Blue HDD
- NAS Storage: 2 × Toshiba N300 4 TB
- RAID Layout: RAID 1 (mirror)

**Notes**
- Runs on bare metal
- Already operational as the main storage node
- Current priorities are backup design, recovery planning, and data protection practices
- Main data storage is mirrored, but backup strategy still needs to be defined
- Future storage expansion should be planned separately

---

### atlas

**Role**  
Media server

**Operating System**  
Ubuntu Server 24.04

**Purpose**  
Run Jellyfin and serve media clients

**Hardware**
- CPU: Intel Core i5-7300HQ (4 cores)
- Memory: 8 GB RAM
- GPU: NVIDIA GeForce GTX 1050 Mobile
- Integrated graphics: Intel HD Graphics 630
- Storage: 500 GB SSD (~439 GB usable)

**Notes**
- Laptop hardware; Jellyfin runs here in Docker
- Media storage migration to `vault` is complete: the library is mounted over SMB
- Hostname may be renamed in the future to better reflect its service role

---

### rpi-01

**Role**  
Edge node: internal DNS, WireGuard VPN endpoint, reverse proxy, Cloudflare tunnel

**Operating System**  
Ubuntu Server 24.04.4 (kernel `6.8.0-1057-raspi`, `aarch64`)

**Purpose**  
Serve core network functions at the edge of the homelab; secondarily support experimentation and future orchestration work

**Hardware**
- Model: Raspberry Pi 5
- Memory: 16 GB RAM
- Storage: 512 GB NVMe (~477 GB usable)

**Notes**
- Runs AdGuard Home, nginx, WireGuard, and cloudflared
- Load-bearing for DNS and VPN; not a disposable lab node anymore
- Very low resource usage, so there is headroom for more workloads
- Still the intended first node for future cluster work

---

### Personal PC

**Role**  
Workstation

**Purpose**  
Development, administration, documentation, and active project work

**Hardware**
- CPU: Intel Core i9-14900KF
- Memory: 64 GB DDR5 RAM
- GPU: NVIDIA GeForce RTX 5070Ti
- Storage: 2 TB Kingston SFYRD NVMe

**Notes**
- Main management and development machine
- Supports the homelab operationally but is not treated as part of the core infrastructure

## Hardware Planning Notes

The homelab is currently in a transitional stage. Some hardware roles are already defined, while others are still being refined based on real workload behavior and infrastructure priorities.

Current hardware decisions are guided by the following principles:

- keep storage stable and separate from volatile workloads
- reserve stronger systems for virtualization and compute tasks
- avoid unnecessary complexity on older hardware
- document constraints before scaling services further

## Pending Information

The following details should be completed as the hardware inventory is refined:

- hostnames for any future nodes
- network interfaces
- power characteristics
- rack placement