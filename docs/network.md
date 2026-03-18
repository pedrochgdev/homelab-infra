# Network

## Summary

The homelab currently operates on a single flat LAN with static addressing for infrastructure nodes and a separate DHCP pool for non-lab devices. Network segmentation has not yet been introduced, but future work is expected to include VLAN design and more formal network separation.

The network is currently intended for internal use only. No public exposure is configured at this time.

## Primary Network

- Network: `192.168.1.0/24`
- Gateway / Router: `192.168.1.1`
- DNS: router / ISP-provided DNS
- DHCP Pool: `192.168.1.100` to `192.168.1.199`
- Addressing for homelab nodes: static manual IP assignment

## Addressing Scheme

The current addressing plan follows simple functional ranges:

- `192.168.1.10` — network switch management
- `192.168.1.20` to `192.168.1.29` — servers
- `192.168.1.30` onward — Raspberry Pi and future cluster-related expansion
- `192.168.1.50` onward — virtual machines

This convention is intentionally simple and is designed to remain understandable while the environment grows.

## Infrastructure IP Plan

| Hostname | IP Address | Role |
|---|---|---|
| `switch` | `192.168.1.10` | Managed switch |
| `virt` | `192.168.1.20` | Proxmox host |
| `vault` | `192.168.1.21` | NAS |
| `atlas` | `192.168.1.22` | Jellyfin media server |
| `rpi-01` | `192.168.1.30` | Raspberry Pi lab node |
| `monitor` | `192.168.1.50` | Monitoring VM |
| `synthia` | `192.168.1.51` | AI workload VM |

## Physical Network Infrastructure

### Switch

- Model: TP-Link SG3428
- Management IP: `192.168.1.10`
- Role: central switching for the homelab rack and connected infrastructure

The switch is part of the documented infrastructure even though the network is still relatively simple. It will become more important as the environment grows, especially once VLANs or more structured segmentation are introduced.

## Current Topology

The current topology is straightforward:

- all nodes are connected to a single LAN
- no VLANs are configured
- no public ingress is configured
- all services are exposed internally only
- remote access is currently handled indirectly through the personal workstation

## Access Model

The homelab is not currently exposed directly to the public internet.

Current remote access behavior:

- the personal workstation is reachable through Tailscale
- remote desktop access to the workstation is used as an administrative entry point
- from the workstation, internal LAN services can be reached as needed

This approach avoids exposing infrastructure services directly while still allowing remote access when required.

## Internal Service Exposure

The following services are currently intended for internal access only:

- SMB shares on `vault`
- Jellyfin on `atlas`
- Prometheus and Grafana on `monitor`
- Ollama service on `synthia`
- Proxmox management on `virt`

## Current Limitations

The current network design is intentionally simple, but it has several limitations:

- no network segmentation
- no VLAN isolation
- no dedicated management network
- no internal DNS beyond router-provided resolution
- no direct VPN entry point into the homelab itself

## Planned Network Improvements

The following improvements are planned as the environment matures:

- introduce VLANs for more structured separation
- create a more professional network layout
- improve remote access design
- evaluate WireGuard as a direct tunnel into the homelab
- formalize service exposure and access boundaries
- expand network documentation as topology becomes more structured