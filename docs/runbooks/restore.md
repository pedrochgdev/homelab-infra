# Restore

## Summary

This document defines the restoration philosophy for the homelab and outlines how recovery should be approached once backup workflows are in place.

At present, restore capability is not yet operationally mature because a full backup policy has not been finalized. This document exists to establish expectations and the future structure of recovery procedures.

## Recovery Principles

Restoration should follow these principles:

- prioritize the simplest path to service recovery
- distinguish between restoring data and rebuilding services
- keep critical data recovery separate from replaceable data
- document practical procedures before they are needed in an incident

## Recovery Categories

Recovery needs differ depending on what failed.

### Scenario 1: Accidental File Deletion

Examples:
- user deletes files from a share
- media directory is modified unintentionally
- configuration file is removed

Desired response:
- restore only the affected files or directories
- avoid full-service rebuild when unnecessary

### Scenario 2: Data Corruption

Examples:
- corrupted files on a share
- damaged configuration
- partial write issues

Desired response:
- restore known-good version of affected data
- verify integrity after restoration

### Scenario 3: Single Data Disk Failure on `vault`

Examples:
- one drive in RAID 1 fails
- degraded array state

Desired response:
- replace failed drive
- rebuild array
- avoid unnecessary data restoration if redundancy preserved availability

Important note:
RAID rebuild is not the same as restore. If data remains intact, this is an array maintenance task, not a backup restore task.

### Scenario 4: NAS Host Failure

Examples:
- operating system failure
- boot disk failure
- motherboard or system failure
- unrecoverable host-level corruption

Desired response:
- rebuild host
- reassemble storage if possible
- restore configuration and protected data as needed

### Scenario 5: Service Host Failure

Examples:
- `atlas` fails
- `virt` fails
- VM loss or corruption

Desired response:
- rebuild service host
- restore service configuration
- reconnect service to centralized storage if applicable

## Recovery Priorities

When multiple systems are affected, recovery should prioritize:

1. access to critical personal data
2. storage node availability
3. infrastructure visibility and monitoring
4. core application services
5. lower-priority media or experimental workloads

## Restore Requirements

A complete restore capability should eventually include:

- known backup source
- clear mapping of what data is protected
- location of relevant configuration backups
- restore commands or procedures
- post-restore validation steps

## Current Limitations

The homelab does not yet have:

- a finalized backup strategy
- restore-tested backup sets
- documented service rebuild procedures
- formal recovery time expectations
- retention policy aligned with restore objectives

## Minimum Future Restore Scope

The following areas should eventually have explicit restore procedures:

- `vault` share recovery
- Samba configuration recovery
- `mdadm` RAID reassembly workflow
- Jellyfin configuration restore
- VM configuration restore
- monitoring stack restore
- critical personal directory restore

## Post-Restore Validation

Any restore procedure should validate:

- expected files are present
- ownership and permissions are correct
- services can read restored data
- storage mounts are correct
- clients can access the restored resource
- application behavior is normal

## Immediate Next Steps

The next practical recovery improvements should be:

1. define what data is actually backed up
2. create a restore checklist for `vault`
3. create a rebuild checklist for `atlas`
4. create a rebuild checklist for `virt`
5. perform at least one small restore test and document the outcome

## Notes

This document is intentionally procedural in tone but still incomplete in implementation detail. It should become more specific once real backup workflows and validation routines are in place.