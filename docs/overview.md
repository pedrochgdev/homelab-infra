# Overview

## Summary

This homelab is a personal infrastructure environment built for practical learning, experimentation, and service hosting. It combines stable services with exploratory workloads, with an emphasis on documenting architectural decisions, maintaining clear node roles, and building an environment that can grow over time.

The current design is centered around the following functional areas:

- virtualization
- storage
- observability
- media services
- AI workloads
- experimentation and future orchestration
- cybersecurity-oriented learning

## Objectives

The main objectives of the homelab are:

- build practical experience with self-hosted infrastructure
- maintain a structured environment for testing and iteration
- separate stable services from experimental workloads
- centralize storage for media and future shared data
- document architecture, changes, and operational procedures
- create a platform for learning systems administration, networking, and security-related topics

## High-Level Architecture

### virt

`virt` is the primary virtualization host and currently runs Proxmox. It is the main compute node for infrastructure virtual machines and selected service workloads.

At present, it hosts at least two virtual machines:

- `synthia`, which runs AI-related workloads and hosts an Ollama model
- `monitor`, which runs Prometheus and Grafana for infrastructure monitoring

This node is the main platform for virtualization and service consolidation.

### vault

`vault` is the dedicated NAS node and runs Ubuntu Server on bare metal. Its role is to provide centralized storage for media and future shared data across the homelab.

The NAS is already operational. It uses a dedicated OS disk and a mirrored RAID 1 storage array built with two 4 TB Toshiba N300 drives for data storage.

Current work is focused on data protection strategy, backup planning, recovery procedures, and the migration of media storage so Jellyfin no longer depends on local storage.

This node is intentionally separated from the main AI workload environment in order to keep storage stable and predictable.

### atlas

`atlas` currently serves as the media node and runs Jellyfin on Ubuntu Server. It is an active service node, but its media library is expected to be moved to `vault` so that storage and application hosting are properly separated.

The service layer may remain on `atlas` or be moved later if needed.

### rpi-01

`rpi-01` is a Raspberry Pi 5 with 16 GB RAM and NVMe storage. It is an active lab node intended for experimentation, lightweight services, and future orchestration or Kubernetes-related work.

It should not be treated only as a placeholder for a future cluster. It is part of the current lab and can be used independently while the broader cluster design is still being defined.

### Personal workstation

The personal workstation is used for development, administration, documentation, and active project work. It supports the homelab but is not treated as a core infrastructure node.

## Design Approach

The homelab is being developed incrementally rather than as a fixed final architecture. This is intentional. The environment exists both to provide useful services and to support learning through practical deployment, iteration, and redesign.

Roles are defined clearly enough to maintain operational structure, but the overall system remains flexible so it can evolve based on workload behavior, project goals, and future hardware decisions.

## Current Priorities

The current infrastructure priorities are:

1. formalize backup and recovery strategy for `vault`
2. move media storage from `atlas` to `vault`
3. continue using `virt` as the main virtualization node
4. document node roles, services, and dependencies more formally
5. assign practical workloads to `rpi-01` while preserving its future role in cluster experimentation
6. keep the homelab usable as a learning platform for infrastructure and security topics

## Architectural Notes

- Storage should remain separate from the most experimental AI workloads.
- Stable services should avoid unnecessary coupling to active development systems.
- Monitoring should remain centralized and independent of individual application nodes.
- The repository should act as the source of truth for infrastructure layout, node roles, and operating procedures.
- Infrastructure should remain understandable and rebuildable without excessive tooling.

## Scope Boundaries

This repository is focused on infrastructure, service placement, operational documentation, and system organization. It is not intended to store secrets, large media assets, or private runtime data.