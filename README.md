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

.
├─ README.md
├─ docs/
│  ├─ overview.md
│  ├─ hardware.md
│  ├─ network.md
│  ├─ nodes/
│  ├─ services/
│  ├─ runbooks/
│  └─ roadmap.md
├─ inventory/
├─ configs/
├─ scripts/
└─ diagrams/

## Documentation Areas

- `docs/overview.md`: high-level architecture and design decisions
- `docs/hardware.md`: hardware inventory and node capabilities
- `docs/network.md`: logical network layout, addressing, and connectivity
- `docs/nodes/`: node-specific documentation
- `docs/services/`: service-specific documentation
- `docs/runbooks/`: operational procedures
- `inventory/`: infrastructure inventory and addressing plans
- `configs/`: sanitized configuration references and examples
- `scripts/`: maintenance and automation scripts
- `diagrams/`: topology and rack diagrams

## Status

This homelab is under active development. Core services are already operational, while parts of the environment are still being refined, expanded, or reorganized.

## Notes

This repository does not include secrets, private credentials, or sensitive local configuration values. Any configuration examples are sanitized before publication.