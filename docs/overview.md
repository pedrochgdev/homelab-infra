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

At present, it hosts three virtual machines:

- `synthia` (VMID 100, named `ia` in Proxmox), which runs AI-related workloads and hosts an Ollama model
- `monitor` (VMID 101), which runs Prometheus and Grafana for infrastructure monitoring
- `dns` (VMID 102), which runs a secondary AdGuard Home resolver

This node is the main platform for virtualization and service consolidation.

### vault

`vault` is the dedicated NAS node and runs Ubuntu Server on bare metal. Its role is to provide centralized storage for media and future shared data across the homelab.

The NAS is already operational. It uses a dedicated OS disk and a mirrored RAID 1 storage array built with two 4 TB Toshiba N300 drives for data storage.

The media migration is complete: Jellyfin on `atlas` now consumes the library over SMB rather than local disk. The backup tiers defined in the runbooks are implemented, with `vault` hosting the local restic repository, an off-site copy on an external drive, and nightly sweeps of the staged dumps; the remaining data-protection work is the tier 2 restore test and keeping the off-site drive out of the house.

This node is intentionally separated from the main AI workload environment in order to keep storage stable and predictable.

### atlas

`atlas` serves as the media node and runs Jellyfin in Docker on Ubuntu Server. Its media library now lives on `vault` and is mounted over SMB, so storage and application hosting are properly separated.

The service layer may remain on `atlas` or be moved later if needed.

### rpi-01

`rpi-01` is a Raspberry Pi 5 with 16 GB RAM and NVMe storage. It has become the edge node of the homelab: it serves internal DNS through AdGuard Home, terminates the WireGuard VPN, runs the nginx reverse proxy that fronts internal services, and hosts the Cloudflare tunnel used to publish one personal project externally.

This makes it load-bearing rather than experimental. Cluster and Kubernetes work remain a longer-term goal for this node, but its current role is operational, and the concentration of edge functions on a single host is a known architectural weakness.

### Personal workstation

The personal workstation is used for development, administration, documentation, and active project work. It supports the homelab but is not treated as a core infrastructure node.

## Design Approach

The homelab is being developed incrementally rather than as a fixed final architecture. This is intentional. The environment exists both to provide useful services and to support learning through practical deployment, iteration, and redesign.

Roles are defined clearly enough to maintain operational structure, but the overall system remains flexible so it can evolve based on workload behavior, project goals, and future hardware decisions.

## Current Priorities

The current infrastructure priorities are:

1. restore GPU acceleration on `synthia`, which currently runs inference on CPU
2. finish proving the backup strategy — tier 1 restored cleanly from the local repository on 2026-08-13, but tier 2 and the off-site drive remain untested
3. reduce the concentration of edge roles on `rpi-01` and isolate public ingress from internal services
4. define an IPv6 posture, since every node holds a globally routable address
5. continue using `virt` as the main virtualization node
6. document node roles, services, and dependencies more formally
7. keep the homelab usable as a learning platform for infrastructure and security topics

## Architectural Notes

- Storage should remain separate from the most experimental AI workloads.
- Stable services should avoid unnecessary coupling to active development systems.
- Monitoring should remain centralized and independent of individual application nodes.
- The repository should act as the source of truth for infrastructure layout, node roles, and operating procedures.
- Infrastructure should remain understandable and rebuildable without excessive tooling.

## Scope Boundaries

This repository is focused on infrastructure, service placement, operational documentation, and system organization. It is not intended to store secrets, large media assets, or private runtime data.