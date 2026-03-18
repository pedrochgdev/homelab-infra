# NAS

## Summary

The NAS service is hosted on `vault` and provides centralized storage for the homelab. It is already operational and currently serves as the primary shared storage node for media, personal data, family-shared data, and future backups.

The current implementation is intentionally simple and Linux-native: Ubuntu Server on bare metal, `mdadm` RAID 1, `ext4`, and SMB shares.

## Host

- Node: `vault`
- Platform: Ubuntu Server `24.04`

## Purpose

The NAS is intended to serve the following functions:

- centralized media storage
- shared file storage
- personal storage
- future backup storage
- future protected storage workflows for the homelab

## Storage Architecture

### OS Storage

- 1 TB WD Blue HDD

### Data Storage

- 2 × Toshiba N300 4 TB
- RAID level: RAID 1
- RAID method: `mdadm`
- Array device: `/dev/md0`
- Filesystem: `ext4`
- Label: `nas_raid1`
- Mount point: `/srv/nas`

## Design Principles

The NAS currently follows a straightforward design:

- separate operating system disk from data storage
- use mirrored storage for resilience against single-drive failure
- keep the filesystem and share structure simple
- prioritize operational clarity over unnecessary abstraction
- treat the NAS as both a production-like storage node and a learning platform

## Directory Layout

Primary storage structure:

- `/srv/nas/backups`
- `/srv/nas/content`
- `/srv/nas/content/media`
- `/srv/nas/shared`
- `/srv/nas/users`

Current media structure includes:

- `/srv/nas/content/media/anime`
- `/srv/nas/content/media/donghuas`
- `/srv/nas/content/media/minidramas`
- `/srv/nas/content/media/movies`
- `/srv/nas/content/media/music`
- `/srv/nas/content/media/series`

User-specific storage currently includes:

- `/srv/nas/users/drocho`

## Access Method

The NAS is currently exposed through SMB only.

This is the active and documented access method for:

- media storage
- personal storage
- shared storage
- backup destination planning

## Current SMB Shares

### `media`
- Path: `/srv/nas/content/media`

### `personal`
- Path: `/srv/nas/users/drocho`

### `family`
- Path: `/srv/nas/shared/family`

### `backups`
- Path: `/srv/nas/backups`

## Share Access Policy

Current share configuration characteristics:

- authenticated access
- only one active user at present: `drocho`
- read/write enabled
- group-based permission model using `nas`
- inherited permissions enabled

Current intent:
- keep the system private and controlled while learning and refining the storage model
- potentially expand access later once permissions, risk boundaries, and share design are better defined

## Current State

The NAS is already functional and in daily use as a storage node.

However, operational maturity is still incomplete in the following areas:

- no defined backup policy yet
- no documented recovery workflow yet
- no SMART monitoring yet
- no formal disk health alerting yet
- no tested incident response process yet
- media has not yet been migrated from `atlas`

## Data Protection Notes

The current RAID 1 array improves resilience, but it does not replace backup.

Important distinction:

- RAID 1 protects against a single disk failure
- RAID 1 does not protect against deletion, corruption, ransomware, configuration mistakes, or host-level failure

A real protection model still needs to define:
- what data is critical
- what data can be recreated
- what data needs versioned backup
- what recovery objective is acceptable for each data category

## Planned Improvements

The following items are current priorities:

- move Jellyfin media storage from `atlas` to `vault`
- define backup policy by directory or data type
- introduce SMART monitoring and storage health checks
- document recovery procedures
- improve storage security posture
- refine permission and access model before expanding to additional users

## Risk Areas

Current known risk areas include:

- absence of independent backups
- absence of storage health monitoring
- lack of formal recovery testing
- single-user operational model without broader access controls
- incomplete separation between active storage and protected backup strategy