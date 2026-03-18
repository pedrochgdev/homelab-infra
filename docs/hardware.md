# Hardware

## Summary

This document tracks the current hardware assigned to the homelab and its intended role within the environment. It is intended to provide a concise operational inventory rather than exhaustive benchmark-level detail.

## Node Inventory

### virt

**Role**  
Primary virtualization host

**Operating System**  
Proxmox VE 9.1.0

**Purpose**  
Run virtual machines and supporting infrastructure services

**Hardware**
- CPU: Intel Core i7-9700K
- Memory: 16 GB DDR4 RAM
- GPU: NVIDIA GeForce GTX 2060
- Storage: 500 GB WD Blue SSD

**Current Virtual Machines**
- `synthia`: AI workload VM hosting an Ollama model
- `monitor`: monitoring VM running Prometheus and Grafana

**Notes**
- Main compute node for virtualization
- Current AI workload behavior suggests VRAM is a more significant constraint than system RAM for the existing Ollama deployment
- VM allocation and storage layout should be documented separately

---

### vault

**Role**  
Dedicated NAS server

**Operating System**  
Ubuntu Server 24.04

**Purpose**  
Provide centralized network storage for media and future shared data

**Hardware**
- CPU: Intel Core i7-3378
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
- CPU: Intel Core i5-7300HQ
- Memory: 8 GB RAM
- GPU: NVIDIA GeForce GTX 1060
- Storage: 500 GB SSD

**Notes**
- Jellyfin is currently hosted here
- Media storage is expected to move to `vault` over the network
- Hostname may be renamed in the future to better reflect its service role

---

### rpi-01

**Role**  
Active lab node for experimentation and future cluster use

**Operating System**  
Ubuntu Server 24.04

**Purpose**  
Support experimentation, lightweight services, and future Kubernetes or orchestration work

**Hardware**
- Model: Raspberry Pi 5
- Memory: 16 GB RAM
- Storage: 512 GB NVMe

**Notes**
- First planned node in a future cluster-oriented lab environment
- Can be used independently before a broader cluster is built
- Intended for experimentation and learning

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
- current operating system versions
- exact laptop memory capacity
- personal workstation hardware details if needed for documentation