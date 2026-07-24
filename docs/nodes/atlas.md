# atlas

## Summary

`atlas` is the media server node in the homelab. It runs Jellyfin on Ubuntu Server through Docker.

The migration of media storage from local disk to the NAS on `vault` is complete:
the library is now mounted over SMB from `vault` and no longer consumes local
storage. `atlas` hosts the application; `vault` holds the data.

## Role

- Jellyfin media server
- media playback and transcoding node

## Operating System

- Platform: Ubuntu Server
- Version: `24.04`

## Hardware

- CPU: Intel Core i5-7300HQ (4 cores)
- Memory: 8 GB RAM
- GPU: NVIDIA GeForce GTX 1050 Mobile
- Integrated graphics: Intel HD Graphics 630
- Local Storage: 500 GB SSD (~439 GB usable)

## Service Stack

- Docker version: `29.1.2`, Compose v5.1.1
- Deployment style: Docker-based, one compose stack per service under `/srv`

| Service | Port | Purpose |
|---|---|---|
| Jellyfin | `8096` | Media server, NAS-backed library |
| Vaultwarden | `8222` | Password manager, TLS terminated by the container |

Vaultwarden is documented separately in
[`docs/services/vaultwarden.md`](../services/vaultwarden.md). It was placed here
because Docker was already running, the node had spare capacity, and it is
internal-only.

## Storage Layout

Container bind mounts:

| Host path | Container path | Backing |
|---|---|---|
| `/mnt/media_nas` | `/media` | SMB share `//192.168.1.21/media` on `vault` |
| `/srv/jellyfin/config` | `/config` | Local disk |
| `/srv/jellyfin/cache` | `/cache` | Local disk |

The compose file lives at `/srv/jellyfin/docker-compose.yml`.

### NAS Mount

The media share is mounted through a systemd automount (`x-systemd.automount`)
rather than a static `fstab` entry, so the mount is established on first access
instead of at boot. Relevant mount options:

- protocol: CIFS `3.1.1`
- SMB user: `jellyfin_smb` (a dedicated account, distinct from `drocho`)
- `soft` — I/O fails rather than hanging indefinitely if `vault` is unreachable
- `uid`/`gid` forced to `1000` so the container sees consistent ownership

Configuration and cache remain on local disk deliberately, so Jellyfin metadata
does not depend on NAS availability.

## Jellyfin Behavior

- Default port configuration
- Hardware acceleration enabled
- GPU-assisted transcoding in use
- Single-user consumption pattern at present

## Access Model

Jellyfin is intended for internal LAN and VPN access. It listens on port `8096`
and is also reachable through the reverse proxy on `rpi-01`.

There is no public exposure of this service. Remote access is available through
the WireGuard VPN on `rpi-01`, or indirectly through the personal workstation
using Tailscale and remote desktop.

## Current State

The service is active and functional, and has been running continuously for
months. Media storage is centralized on `vault` as intended.

## Operational Notes

- content acquisition and media workflow are still relatively manual
- the host may be renamed later to better reflect its service role
- Docker holds an unused `infra_net` network, suggesting a stack defined and
  never deployed
- a reboot is pending: the running kernel is `6.8.0-106` while `6.8.0-136` is
  installed, so several months of kernel patches are staged but inactive

### Leftover local media

`/srv/media` still holds 126 GB from before the NAS migration. Jellyfin no longer
reads it — the container mounts `/mnt/media_nas` instead — so it is invisible to
the service while still consuming disk.

It cannot simply be deleted. A file-level comparison against the NAS library
found **14 files present only on `atlas`**, totalling 29 GB. Those files sit on
an unmirrored SSD, outside the RAID array, outside every backup, and unreachable
by Jellyfin.

The safe sequence is to copy those 14 files to `vault`, verify them, and only
then remove `/srv/media`, which would reclaim 126 GB and drop disk usage from
36 percent to roughly 5 percent.

## Planned Changes

- consolidate the 14 orphaned media files onto `vault` and reclaim `/srv/media`
- reboot to activate the pending kernel
- replace the Vaultwarden self-signed certificate with a Let's Encrypt one
- verify hardware transcoding still behaves correctly against NAS-backed media
- confirm playback behavior when `vault` is unavailable, given the `soft` mount
- document the compose stacks in `configs/`