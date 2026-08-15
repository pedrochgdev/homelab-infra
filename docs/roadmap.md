# Roadmap

## Summary

This roadmap tracks the current priorities and longer-term direction of the homelab. The environment is already operational, but several areas still need to mature in order to make it more structured, resilient, and professionally organized.

The roadmap reflects both practical needs and learning goals.

## Current Priorities

The highest-priority items at this stage are:

1. take the external off-site drive out of the house — until it lives elsewhere
   there is still no copy that survives fire or theft, only a third one in the
   same room
2. fit a UPS to `vault` and set its BIOS to power on after AC loss, after it
   stopped dead for six days and took both backup tiers with it
3. close the inbound IPv6 firewall rule on the router; the host side was closed
   on 2026-08-13
4. deploy Alertmanager for the metrics Prometheus already collects — the backup
   alerts no longer depend on it, since failure and staleness alerting now runs
   through `ntfy`
4. audit every service bound to `::` against the router's IPv6 policy, starting with Samba on `vault` and the Proxmox interface on `virt`
5. restore GPU acceleration on `synthia`, which currently falls back to CPU inference
6. reduce the concentration of edge roles on `rpi-01`, which now carries DNS, VPN, reverse proxy, and public ingress
7. continue refining infrastructure documentation
8. improve cable management in the rack
9. continue using the environment as a platform for AI, automation, infrastructure, and security learning

### Recently completed

- media storage migrated from `atlas` to `vault` over SMB
- SMART and RAID health monitoring deployed on `vault` and scraped by `monitor`
- `rpi-01` assigned a real workload: internal DNS, VPN, and reverse proxy
- WireGuard deployed as a direct entry point into the homelab
- secondary DNS instance deployed on the `dns` VM, with failover tested
- retired the DuckDNS vhost on `rpi-01` and added an explicit catch-all vhost,
  closing the direct serving path from the internet to the workstation
- all three backup tiers implemented: daily restic to local and Backblaze B2,
  staged dumps from `atlas` and `rpi-01`, and weekly `vzdump` of the VMs
- Vaultwarden deployed on `atlas`, backed up nightly from day one
- internal CA deployed, with TLS termination on `rpi-01` for
  `pass.home.arpa` and the root certificate served at `ca.home.arpa`

## Short-Term Goals

### Storage and Data Protection

The strategy is defined in [`docs/runbooks/backups.md`](runbooks/backups.md).
All three tiers run: the off-site tier is now an external disk (first verified
copy 2026-08-13) after B2 was set aside on cost preference. The tier 1 restore
test passed locally on 2026-08-13.

- take the external drive out of the house, and keep the repository password on
  paper away from it
- restore a VM to a scratch VMID and boot it, for the tier 2 test
- give `vault` a UPS so the next power event ends in a clean shutdown
- remove the dead B2 leg remnants (bucket and scoped key still exist)

### Alerting

Prometheus collects metrics that nothing acts on: there is no `alertmanager`
and no `rule_files`. This blocks failure detection for storage and backups alike.

- deploy Alertmanager alongside Prometheus on `monitor`
- turn the existing SMART and RAID metrics into alerts
- alert when the most recent successful backup is older than 48 hours
- route notifications somewhere that reaches a phone

### Raspberry Pi
- configuration backup is in place (nightly at 02:20 into the staging area on `vault`)
- DNS failover to the `dns` VM has been tested and works
- decide whether public ingress should move off this node
- keep cluster experimentation as a longer-term goal

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
- use the DuckDNS hostname as the WireGuard client `Endpoint` so the tunnel
  recovers on reconnect after an ISP address change, instead of requiring a
  manual lookup and config edit
- confirm what actually refreshes the DuckDNS record, since nothing on `rpi-01` does
- document WireGuard peers and define a key rotation practice
- confirm client profiles set `DNS` to AdGuard, or `.home.arpa` names fail to
  resolve away from home even with the tunnel up
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

- whether the edge tier on `rpi-01` should be split so public ingress no longer shares a host with internal DNS and VPN
- whether `vault` should stop being both the backup destination and the NFS host for tier 2, since losing it stops both tiers at once
- how should storage backup policy vary between media, personal files, shared data, and infrastructure backups
- whether SMB remains the preferred long-term method for serving media to `atlas`
- how far to push network segmentation and service isolation in the current hardware environment
- whether IPv6 should be disabled at the router or planned and firewalled deliberately

## Strategic Direction

The homelab is not intended to be static. Its value comes from being both useful and educational.

The long-term direction is to keep building a more capable and more professional environment while preserving the ability to experiment, learn, redesign, and iterate.