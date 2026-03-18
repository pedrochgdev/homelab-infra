# Roadmap

## Summary

This roadmap tracks the current priorities and longer-term direction of the homelab. The environment is already operational, but several areas still need to mature in order to make it more structured, resilient, and professionally organized.

The roadmap reflects both practical needs and learning goals.

## Current Priorities

The highest-priority items at this stage are:

1. move Jellyfin media storage from `atlas` to `vault`
2. define data protection, backup, and recovery strategy for the NAS
3. assign a practical workload to `rpi-01`
4. continue refining infrastructure documentation
5. improve cable management in the rack
6. continue using the environment as a platform for AI, automation, infrastructure, and security learning

## Short-Term Goals

### Storage and Data Protection
- move media storage to `vault`
- define backup policy by directory or data class
- document risk areas for storage
- define prevention and recovery actions
- introduce storage monitoring and health checks

### Raspberry Pi
- assign an initial purpose to `rpi-01`
- begin using it as an active experimentation node instead of leaving it idle
- explore suitable lightweight workloads before cluster expansion

### Observability
- improve Grafana presentation and structure
- make dashboards more organized and professional
- continue refining visibility into the infrastructure

### Rack and Physical Organization
- improve cable management
- continue cleaning up and organizing the rack layout
- document the physical environment more clearly over time

## Mid-Term Goals

### Network Maturity
- experiment with the network in a more structured way
- design and introduce VLANs
- move toward a more professional network layout
- better define access boundaries and service placement

### NAS Maturity
- formalize SMB access policy
- expand beyond the current single-user model if needed
- define backup retention and restore expectations
- improve operational safety around storage administration

### Virtualization
- continue using `virt` as the main VM platform
- add a Minecraft server VM
- improve the overall structure of virtualized services

### Remote Access
- evaluate WireGuard as a more direct homelab access model
- improve remote administration design over time

## Long-Term Goals

### Cluster and Orchestration
- expand beyond `rpi-01`
- build out a future cluster environment
- experiment with orchestration and cluster design
- use the Raspberry Pi platform as a foundation for more advanced distributed infrastructure learning

### Security and Cybersecurity Practice
- use the homelab as a practical platform for cybersecurity learning
- create safer environments for experimentation
- improve segmentation, access control, and service isolation
- develop a more professional operational mindset around risk and exposure

### AI and Automation
- continue using the environment to support AI experimentation
- refine the infrastructure around the LLM-serving layer
- expand learning around automation and operational workflows

## Open Questions

The following architectural questions are still open:

- what should `rpi-01` do before a larger cluster exists
- what is the best long-term access method between LAN, Tailscale-based workflow, and a future WireGuard tunnel
- how should storage backup policy vary between media, personal files, shared data, and infrastructure backups
- whether SMB remains the preferred long-term method for serving media to `atlas`
- how far to push network segmentation and service isolation in the current hardware environment

## Strategic Direction

The homelab is not intended to be static. Its value comes from being both useful and educational.

The long-term direction is to keep building a more capable and more professional environment while preserving the ability to experiment, learn, redesign, and iterate.