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
- two accounts: `drocho` for interactive use and `jellyfin_smb` as a service
  account used by `atlas` to mount the media share
- both belong to the `nas` group (GID 1001)
- read/write enabled
- group-based permission model using `nas`
- inherited permissions enabled

Current intent:
- keep the system private and controlled while learning and refining the storage model
- potentially expand access later once permissions, risk boundaries, and share design are better defined

## Current State

The NAS is already functional and in daily use as a storage node.

The RAID 1 array is healthy, with both members in sync. Roughly 816 GB of 3.6 TB
is in use, broken down as 769 GB of content, 39 GB of user data, and 9.5 GB of
backups.

Media migration from `atlas` is complete, and disk health monitoring is running:
`smartmontools`, `mdmonitor`, and `smartctl_exporter` on port `9633`, scraped by
`monitor`.

Backups are implemented and running on schedule; see
[`docs/runbooks/backups.md`](../runbooks/backups.md).

Operational maturity is still incomplete in the following areas:

- no restore test has ever been performed
- no alerting rules on top of the collected SMART metrics
- no tested incident response process yet

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

The backup strategy defined in
[`docs/runbooks/backups.md`](../runbooks/backups.md) is fully implemented: all
three tiers run on schedule, with `/srv/nas/users` and the staging area going
daily to the local repository, and to an off-site external drive whenever it is
plugged in.

The following items are current priorities:

- perform and record the tier 2 restore test
- define alerting rules on top of the existing SMART and RAID metrics
- improve storage security posture
- refine permission and access model before expanding to additional users

## Risk Areas

Current known risk areas include:

- backups exist but have never been restore-tested
- disk health metrics are collected but not alerted on
- narrow access model without broader controls
- the restic repository key lives only on `vault` itself; a copy outside the
  homelab is required for an off-site restore to be possible at all