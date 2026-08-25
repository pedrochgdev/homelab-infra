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
- Version: `9.1.6` (`pve-manager/9.1.6`)
- Base system: Debian 13 "trixie", kernel `6.17.13-2-pve`
- Filesystem: `ext4`

## Hardware

- CPU: Intel Core i7-9700K (8 cores)
- Memory: 16 GB RAM
- GPU: NVIDIA GeForce RTX 2060, passed through to the AI workload VM
- Integrated graphics: Intel UHD Graphics 630, retained by the host
- Host Storage: 500 GB SSD (`sda`, ~466 GB)

## Storage Layout

`virt` uses a single local SSD for the Proxmox host and local VM storage.

| Storage | Type | Size | Usage |
|---|---|---|---|
| `local` | dir | ~62 GB | ISOs, backups; about 22 percent used |
| `local-lvm` | lvmthin | ~369 GB | VM disks; about 13 percent used |

Host volumes are `pve-root` (64 GB, `/`) and `pve-swap` (8 GB), with the
remainder allocated to the `pve-data` thin pool.

There is no ZFS pool on this host, despite `zfs-zed` being present in the
default Proxmox service set.

## Network Configuration

- Primary bridge: `vmbr0`
- Current network design: default bridge-based configuration
- Addressing model: static IP
- Host IP: `192.168.1.20`

At present, no advanced multi-interface or segmented network configuration has been introduced. The host is operating on the main LAN and using the standard bridge model.

## Current Virtual Machines

Three VMs are defined and running.

### VMID 101 — monitor

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

### VMID 100 — ia / synthia

**Role**  
AI workload VM

Note the naming mismatch: the VM is named `ia` in Proxmox, while the guest
hostname is `synthia`. Both names appear across documentation and tooling.

**Resources**
- CPU: 1 socket, 2 cores
- Memory: 4 GB RAM
- Disk: 120 GB
- IP: `192.168.1.51`

**Purpose**
- host Ollama
- serve the LLM used by the avatar-related AI project

**GPU Configuration**
- full GPU passthrough is enabled via `hostpci0: 0000:01:00,pcie=1`
- the guest currently cannot use the GPU: `nvidia-smi` reports a driver/library
  version mismatch, so inference falls back to CPU. See
  [`docs/services/ai-stack.md`](../services/ai-stack.md).

---

### VMID 102 — dns

**Role**  
Secondary internal DNS

**Resources**
- CPU: 1 socket, 1 core
- Memory: 2 GB RAM
- Disk: 32 GB
- IP: `192.168.1.52`

**Purpose**
- run a second AdGuard Home instance as backup for the primary on `rpi-01`
- published internally as `adguard2.home.arpa`

## Current State

`virt` is stable and runs all three VMs without issue. Host memory is the
tightest resource: roughly 10 GB of 16 GB is committed with the current VM set,
which constrains the planned Minecraft VM.

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