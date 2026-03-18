# Backups

## Summary

This document defines the backup direction for the homelab, with an emphasis on practical protection rather than unnecessary complexity.

At present, the homelab has storage redundancy on `vault` through RAID 1, but it does not yet have a complete backup strategy. This document establishes the intended backup model and the immediate next steps required to make data protection meaningful.

## Important Distinction

RAID is not a backup.

Current RAID 1 on `vault` protects against:

- failure of a single data disk

It does not protect against:

- accidental deletion
- file corruption
- ransomware
- permission mistakes
- operating system failure
- array misconfiguration
- host-level failure
- theft or physical damage

Any real backup strategy must assume that the live NAS can still fail in ways RAID does not solve.

## Backup Objectives

The backup strategy should aim to provide:

- protection for data that cannot be easily recreated
- separation between live storage and protected copies
- clear distinction between critical and non-critical data
- recoverability that is simple enough to execute under stress

## Data Classes

The homelab should treat data differently depending on its value and how easily it can be recreated.

### Class 1: Critical Personal Data

Examples:
- personal files
- important documents
- irreplaceable user data
- selected configuration files
- credentials exports stored outside the repository, if any

Protection target:
- highest protection priority
- must exist in more than one location
- should be backed up independently from live NAS storage

### Class 2: Infrastructure Data

Examples:
- service configuration
- Proxmox VM definitions
- exported compose files
- monitoring configuration
- scripts and operational notes not already in Git

Protection target:
- important
- should be backed up regularly
- should be recoverable without rebuilding everything manually

### Class 3: Shared and Family Data

Examples:
- files stored under shared directories
- family share content

Protection target:
- medium to high priority depending on actual content
- protection level should be decided based on whether data is replaceable

### Class 4: Media Library

Examples:
- movies
- anime
- music
- series
- other entertainment media

Protection target:
- lower than personal documents if content can be recreated
- still valuable due to time investment and organization effort
- may justify partial backup depending on capacity and cost

## Current State

Current storage reality:

- `vault` stores live NAS data
- RAID 1 is active on the data array
- no formal backup process is currently in place
- no retention policy is currently defined
- no restore testing has been documented yet

## Recommended Backup Model

The backup strategy should be phased rather than attempting to solve everything at once.

### Phase 1: Configuration and Critical Data

Protect first:
- important personal files
- NAS configuration
- Samba configuration
- `mdadm` configuration
- service compose files
- scripts
- selected application config directories
- any infrastructure data not already safely versioned in Git

### Phase 2: Shared Data

Protect next:
- selected shared content
- family share content
- data that would be painful to rebuild manually

### Phase 3: Media Strategy

Decide explicitly:
- what media is worth backing up
- what media can be re-acquired
- what media should be protected because of effort, metadata, or rarity

## Backup Targets

The future backup strategy should use backup targets separate from the live array.

Preferred characteristics of backup targets:

- independent from the live NAS data array
- not permanently exposed as writable primary storage
- simple enough to verify and restore from
- capacity aligned with the protection goals of each data class

Potential backup target options may include:

- additional external disk storage
- another internal disk used for backup only
- remote backup target
- secondary NAS target
- selected off-site copy for critical files

## Backup Frequency Guidance

Initial guidance by data type:

### Critical Personal Data
- frequent backups
- versioned if possible

### Infrastructure Data
- regular scheduled backup
- especially after meaningful changes

### Shared Data
- scheduled backup depending on usage frequency

### Media
- optional, selective, or lower-frequency backup depending on value and available capacity

## Verification

A backup that has never been checked should not be assumed valid.

Future process should include:

- verification that backup jobs actually completed
- spot-checking restored files
- periodic restore testing for critical paths

## Immediate Next Steps

The next practical backup actions should be:

1. define which directories on `vault` are critical
2. classify data by importance
3. choose at least one independent backup target
4. begin backing up critical personal and infrastructure data first
5. document restore steps after the first backup workflow is in place

## Notes

This document defines the intended backup direction. It should be updated once the actual backup implementation, retention policy, and validation workflow are in place.