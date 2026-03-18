# virt

## Summary

`virt` is the primary virtualization host in the homelab. It runs Proxmox VE and serves as the main compute platform for infrastructure virtual machines and future service expansion.

It is currently stable and operational, hosting the AI workload VM and the monitoring VM.

## Role

- primary virtualization host
- service consolidation node
- future host for additional infrastructure VMs

## Operating System

- Platform: Proxmox VE
- Version: `9.1.0`
- Filesystem: `ext4`

## Hardware

- CPU: Intel Core i7-9700K
- Memory: 16 GB RAM
- GPU: NVIDIA GeForce GTX 2060
- Host Storage: 500 GB WD Blue SSD

## Storage Layout

`virt` uses a single local SSD for the Proxmox host and local VM storage.

Current VM disks are stored on:

- `local-lvm`

## Network Configuration

- Primary bridge: `vmbr0`
- Current network design: default bridge-based configuration
- Addressing model: static IP
- Host IP: `192.168.1.20`

At present, no advanced multi-interface or segmented network configuration has been introduced. The host is operating on the main LAN and using the standard bridge model.

## Current Virtual Machines

### monitor

**Role**  
Infrastructure monitoring VM

**Resources**
- CPU: 1 socket, 1 core
- Memory: 2 GB RAM
- Disk: 25 GB
- IP: `192.168.1.50`

**Purpose**
- Prometheus
- Grafana

---

### synthia

**Role**  
AI workload VM

**Resources**
- CPU: 1 socket, 2 cores
- Memory: 4 GB RAM
- Disk: 120 GB
- IP: `192.168.1.51`

**Purpose**
- host Ollama
- serve the LLM used by the avatar-related AI project

**GPU Configuration**
- full GPU passthrough is enabled

## Current State

`virt` is currently stable and runs both existing VMs without issue.

## Planned Expansion

A future virtual machine is planned for:

- Minecraft server hosting

Additional VMs may be added later depending on infrastructure growth, available memory, and project requirements.

## Operational Notes

- GPU memory is currently a more relevant constraint than system RAM for the active Ollama deployment
- Proxmox is the main platform for service isolation and workload experimentation
- storage for VMs remains local to the host at this stage
- the host currently operates on a simple LAN design without segmentation

## Future Considerations

The following items may be expanded later:

- more formal VM inventory
- backup strategy for VMs
- snapshot policy
- resource allocation standards
- Minecraft server VM deployment
- network segmentation for virtual workloads