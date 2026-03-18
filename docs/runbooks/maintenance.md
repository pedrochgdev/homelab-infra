# Maintenance

## Summary

This document defines general maintenance expectations for the homelab. Its purpose is to keep the environment stable, understandable, and recoverable while still allowing room for experimentation.

Maintenance in this homelab should be deliberate rather than reactive. The goal is not to eliminate experimentation, but to ensure that core services remain manageable over time.

## Maintenance Principles

The homelab should be maintained according to the following principles:

- keep critical services predictable
- separate experimentation from core storage
- document meaningful changes
- verify system health before and after changes
- avoid making multiple infrastructure changes at once without clear rollback paths

## Maintenance Areas

### Storage

Storage maintenance on `vault` should include:

- checking RAID status
- monitoring disk health
- reviewing available capacity
- validating share access
- ensuring permissions remain correct
- reviewing backup and recovery readiness as the backup model matures

### Virtualization

Maintenance on `virt` should include:

- reviewing Proxmox host health
- checking VM status
- confirming sufficient storage and memory headroom
- documenting new VM additions or major resource changes
- reviewing GPU passthrough stability when applicable

### Media Services

Maintenance on `atlas` should include:

- checking Jellyfin availability
- validating media path access
- confirming hardware acceleration behavior when relevant
- reviewing storage migration progress toward `vault`

### Monitoring

Maintenance of `monitor` should include:

- checking Prometheus and Grafana availability
- reviewing dashboards for stale or missing data
- improving dashboard structure over time
- validating exporter coverage as infrastructure grows

### Raspberry Pi Lab Node

Maintenance on `rpi-01` should include:

- keeping the base system updated
- documenting any experimental services introduced later
- avoiding undocumented configuration drift

## Regular Maintenance Tasks

The following routine checks should be part of ongoing maintenance:

- verify host reachability
- verify service availability
- review disk usage
- review RAID health
- review major logs when needed
- review pending infrastructure changes
- document any significant change in the repository

## Change Management Guidance

Before making a significant infrastructure change:

- identify the system being changed
- identify what services depend on it
- decide whether rollback is possible
- confirm whether data is protected
- document the change if it affects architecture, service placement, or recovery behavior

Examples of significant changes:

- moving media storage
- changing NAS permissions
- adding VMs
- modifying Proxmox networking
- introducing VLANs
- enabling new remote access methods

## Physical Maintenance

The homelab also requires physical maintenance.

Current focus areas include:

- improving cable management
- keeping rack organization cleaner
- documenting physical layout over time
- checking airflow and unnecessary hardware where relevant

## Security-Oriented Maintenance

As the homelab becomes a platform for cybersecurity learning, maintenance should also include:

- reviewing exposure boundaries
- avoiding unintended public exposure
- documenting access paths
- reviewing service sprawl
- improving isolation as the network matures

## Current Gaps

Maintenance maturity is still incomplete in the following areas:

- SMART monitoring on `vault`
- formal backup checks
- restore validation
- documented maintenance schedule
- structured change records for major architecture updates

## Immediate Next Steps

The next practical maintenance improvements should be:

1. add storage health monitoring on `vault`
2. improve dashboard quality in Grafana
3. complete the media migration plan from `atlas` to `vault`
4. document future major changes before executing them
5. continue cleaning and organizing the rack physically