# Homelab Infrastructure

This repository contains the infrastructure documentation, service notes, configuration references, and operational planning for my homelab.

## Purpose

This homelab is a personal infrastructure environment built for practical learning, experimentation, and service hosting. It is designed to support hands-on work across multiple areas, including:

- infrastructure and systems administration
- self-hosting
- virtualization
- networked storage
- observability and monitoring
- AI workloads
- media services
- cybersecurity learning and experimentation

The environment is designed to evolve incrementally, with each node assigned a defined role and documented operating context.

## Current Scope

The homelab currently includes the following functional areas:

- virtualization with Proxmox
- centralized network storage
- media services
- AI workloads hosted in virtual machines
- infrastructure monitoring
- experimental services and future cluster work
- documentation and operational planning

## Node Overview

| Node | Hostname | Role | Status |
|---|---|---|---|
| PC1 | `virt` | Proxmox host for infrastructure and virtual machines | Active |
| VM | `synthia` | AI workload VM running Ollama | Active |
| VM | `monitor` | Monitoring VM running Prometheus and Grafana | Active |
| PC2 | `vault` | NAS server running Ubuntu Server on bare metal | Active |
| Laptop | `atlas` | Jellyfin media server | Active |
| Raspberry Pi 5 | `rpi-01` | Active lab node for experimentation and future cluster use | Active |
| Personal PC | N/A | Workstation for development, administration, and project work | Active |

## Architecture Principles

The environment is organized around a few core principles:

- clear separation of roles between nodes
- stable storage independent from experimental workloads
- lightweight, maintainable services
- documentation-first operation
- incremental growth instead of unnecessary complexity
- practical learning through real deployments

## Repository Structure

- `README.md` — repository overview
- `docs/` — architecture, nodes, services, and operational documentation
- `inventory/` — hardware, addressing, and storage inventory
- `configs/` — sanitized configuration references
- `scripts/` — maintenance and automation scripts
- `diagrams/` — topology and rack diagrams

## Documentation Areas

## Key Documentation

- [`docs/overview.md`](docs/overview.md) — high-level architecture and design direction
- [`docs/hardware.md`](docs/hardware.md) — hardware inventory and node capabilities
- [`docs/network.md`](docs/network.md) — network layout, addressing, and access model
- [`docs/nodes/virt.md`](docs/nodes/virt.md) — Proxmox host documentation
- [`docs/nodes/vault.md`](docs/nodes/vault.md) — NAS node documentation
- [`docs/nodes/atlas.md`](docs/nodes/atlas.md) — Jellyfin node documentation
- [`docs/nodes/rpi-01.md`](docs/nodes/rpi-01.md) — Raspberry Pi lab node documentation
- [`docs/nodes/monitor.md`](docs/nodes/monitor.md) — monitoring VM documentation
- [`docs/services/ai-stack.md`](docs/services/ai-stack.md) — AI workload overview
- [`docs/services/jellyfin.md`](docs/services/jellyfin.md) — Jellyfin service documentation
- [`docs/services/nas.md`](docs/services/nas.md) — NAS service documentation
- [`docs/services/minecraft.md`](docs/services/minecraft.md) — planned Minecraft service
- [`docs/runbooks/backups.md`](docs/runbooks/backups.md) — backup planning
- [`docs/runbooks/restore.md`](docs/runbooks/restore.md) — restore and recovery direction
- [`docs/runbooks/maintenance.md`](docs/runbooks/maintenance.md) — maintenance guidance
- [`docs/roadmap.md`](docs/roadmap.md) — current priorities and future plans
- [`inventory/ip-plan.md`](inventory/ip-plan.md) — IP allocation reference
- [`inventory/storage-layout.md`](inventory/storage-layout.md) — storage distribution and migration state

## Status

This homelab is under active development. Core services are already operational, while parts of the environment are still being refined, expanded, or reorganized.

## Notes

This repository does not include secrets, private credentials, or sensitive local configuration values. Any configuration examples are sanitized before publication.