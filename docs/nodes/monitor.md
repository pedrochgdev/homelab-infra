# monitor

## Summary

`monitor` is the infrastructure monitoring VM hosted on `virt`. It runs the current observability stack for the homelab and provides visibility into system health and service behavior.

It is already operational and serves as the central monitoring point for the environment.

## Role

- monitoring VM
- observability node
- dashboard and metrics platform

## Operating System

- Platform: Ubuntu Server
- Version: `24.04`

## Host Platform

- Hosted on: `virt` (VMID 101)
- Virtualization platform: Proxmox VE `9.1.6`

## Resources

- CPU: 1 socket, 1 core
- Memory: 2 GB RAM
- Disk: 25 GB
- IP: `192.168.1.50`

## Service Stack

| Service | Version | Port |
|---|---|---|
| Prometheus | 3.10.0 | `9090` |
| Grafana | 12.4.1 | `3000` |
| nginx | — | `80` |

nginx serves the `display` dashboard from `/var/www/display`, published
internally as `display.home.arpa` through the reverse proxy on `rpi-01`. This is
a custom real-time statistics page for the personal workstation, independent of
the Prometheus and Grafana stack despite sharing the host. It polls a bespoke
agent on `192.168.1.109:9500` that nothing currently monitors. See
[`docs/services/display.md`](../services/display.md).

## Scrape Targets

Prometheus is configured with the following jobs:

| Job | Targets |
|---|---|
| `prometheus` | `localhost:9090` |
| `linux-nodes` | `node_exporter` on the homelab nodes, port `9100` |
| `smartctl_vault` | `192.168.1.21:9633` — disk health on `vault` |
| `windows_desktop` | `192.168.1.109:9182` — personal workstation |
| `nvidia_desktop` | `192.168.1.109:9835` — workstation GPU metrics |

Note that the personal workstation is an active monitoring target even though it
is documented elsewhere as outside the core infrastructure. Its address sits
inside the DHCP pool, so these targets break if the lease changes.

## Purpose

The purpose of `monitor` is to provide:

- infrastructure visibility
- service monitoring
- dashboard-based operational awareness
- a foundation for improving observability maturity over time

## Current State

`monitor` is active and functioning.

It currently supports the environment at a practical level, but its presentation and operational maturity are still expected to improve.

## Current Improvement Areas

The main current improvement areas are:

- make Grafana dashboards more professional
- improve organization and presentation of monitoring data
- expand observability maturity as the homelab grows

## Notes

This VM is intentionally separate from application hosts so monitoring remains centralized and not tied to a single service node.