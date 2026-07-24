# vault

## Summary

`vault` is the dedicated NAS node in the homelab. It runs Ubuntu Server on bare metal and provides centralized network storage for media and other shared data.

The system is already operational. Current work is focused on improving data protection, backup policy, recovery planning, and long-term storage practices.

## Role

- centralized network storage
- media storage backend
- shared file storage
- future backup target and protected data node

## Operating System

- Platform: Ubuntu Server
- Version: `24.04`

## Hardware

- CPU: Intel Core i7-3770 (8 threads)
- Memory: 8 GB DDR3 RAM
- GPU: None
- OS Storage: 1 TB WD Blue HDD (`sda`, ~931 GB)
- NAS Storage: 2 × Toshiba N300 4 TB (`sdb`, `sdc`)
- RAID Layout: RAID 1 (mirror)

## Storage Design

The NAS separates the operating system from the main data array:

- OS runs on a dedicated 1 TB WD Blue HDD, partitioned with LVM
- user and shared data live on a mirrored RAID 1 array
- RAID is built with `mdadm`
- data filesystem is `ext4`

The OS disk uses separate logical volumes so that log or cache growth in `/var`
cannot fill the root filesystem:

| Volume | Size | Mount |
|---|---|---|
| `vg0-lv_root` | 100 GB | `/` |
| `vg0-lv_var` | 40 GB | `/var` |
| `vg0-lv_swap` | 8 GB | swap |

## RAID Implementation

The current array is built with Linux software RAID using `mdadm`:

- Array device: `/dev/md0`
- RAID level: 1
- Filesystem label: `nas_raid1`
- Mount point: `/srv/nas`

The array was created using `mdadm`, persisted in `/etc/mdadm/mdadm.conf`, and mounted through `/etc/fstab`.

## Filesystem and Mounting

- RAID device: `/dev/md0`
- Filesystem: `ext4`
- Label: `nas_raid1`
- Mount point: `/srv/nas`

The ext4 reserved space was reduced from 5 percent to 1 percent to improve usable capacity on a data-oriented filesystem.

## Directory Layout

Primary layout under `/srv/nas`:

- `/srv/nas/backups`
- `/srv/nas/content`
- `/srv/nas/content/media`
- `/srv/nas/shared`
- `/srv/nas/users`
- `/srv/nas/users/drocho`

Current media subdirectories include:

- `anime`
- `donghuas`
- `minidramas`
- `movies`
- `music`
- `series`

## Access Model

The NAS is currently exposed through SMB.

At present:
- access is authenticated
- two accounts exist: `drocho` for interactive use, and `jellyfin_smb` as a
  dedicated service account used by `atlas` to mount the media share
- both accounts belong to the `nas` group (GID 1001)
- group ownership is centered around the `nas` group

## SMB Shares

Current Samba shares include:

### `media`
- Path: `/srv/nas/content/media`

### `personal`
- Path: `/srv/nas/users/drocho`

### `family`
- Path: `/srv/nas/shared/family`

### `backups`
- Path: `/srv/nas/backups`

Current share behavior includes:
- `browseable = yes`
- `read only = no`
- `valid users = drocho`
- `force group = nas`
- `inherit permissions = yes`

## Current State

`vault` is operational and serving as the main NAS node.

Array and capacity:

- `/dev/md0` is healthy, both members present and in sync (`[2/2] [UU]`)
- 3.6 TB total, roughly 816 GB used (about 23 percent)
- current distribution: content 769 GB, users 39 GB, backups 9.5 GB

Health monitoring is in place:

- `smartmontools` and `mdmonitor` are running
- `smartctl_exporter` publishes disk health metrics on port `9633`
- `monitor` scrapes that exporter through a dedicated `smartctl_vault` job

Media migration from `atlas` is complete; the media share is consumed over SMB.

Still pending:

- backup policy is not yet defined
- recovery procedures are not yet documented
- alerting rules on top of the SMART metrics are not yet defined

## Operational Priorities

The main current priorities for `vault` are:

- define backup policy by data type
- define recovery expectations and procedures
- turn the existing SMART and RAID metrics into actual alerts
- improve storage security and operational discipline

## Risk Notes

RAID 1 provides redundancy against single-drive failure, but it is not a backup.

At present, the system still lacks:
- independent backup copies
- recovery testing
- SMART alerting
- formal incident response for storage faults

## Position in the Homelab

`vault` is intended to function both as:

- a practical learning platform for Linux-based storage administration
- a production-like storage node for the homelab

This dual role is intentional and should be reflected in future documentation and operational decisions.