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

- Docker version: `29.1.2`
- Primary application: Jellyfin
- Deployment style: Docker-based

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

- Jellyfin is the only container currently running, though Docker holds an
  additional network (`infra_net`) suggesting other stacks have been defined
- content acquisition and media workflow are still relatively manual
- the host may be renamed later to better reflect its service role
- local disk usage sits around 146 GB of 439 GB after the media migration

## Planned Changes

- verify hardware transcoding still behaves correctly against NAS-backed media
- confirm playback behavior when `vault` is unavailable, given the `soft` mount
- document the compose stack in `configs/`