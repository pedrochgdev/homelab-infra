# Jellyfin

## Summary

Jellyfin is the current media service in the homelab and runs on `atlas` using Docker. The service is operational and uses hardware acceleration for transcoding.

Media storage has been migrated to `vault`: the library is mounted over SMB and
the application no longer depends on local disk for content.

## Host

- Node: `atlas`
- Platform: Ubuntu Server `24.04`
- Runtime: Docker `29.1.2`

## Purpose

- host personal media library
- provide media playback and streaming
- support hardware-accelerated transcoding

## Current Paths

- Media: `/mnt/media_nas` on the host, mounted into the container as `/media`
- Config: `/srv/jellyfin/config`
- Cache: `/srv/jellyfin/cache`
- Compose file: `/srv/jellyfin/docker-compose.yml`

## Networking

- Port: 8096
- Exposure: internal LAN and VPN, also reachable through the reverse proxy on `rpi-01`

Remote access is over the WireGuard VPN on `rpi-01`.

## Media and Storage

The media library lives on `vault` and is consumed over SMB:

- share: `//192.168.1.21/media`
- protocol: CIFS `3.1.1`
- account: `jellyfin_smb`, a dedicated service user
- mount style: systemd automount, established on first access rather than at boot
- failure mode: `soft`, so I/O returns errors instead of hanging if `vault` is down

Configuration and cache remain on local disk by design, so Jellyfin metadata and
transcode scratch space do not depend on NAS availability.

## Transcoding

- hardware acceleration enabled
- GPU-assisted transcoding in use

## Usage Pattern

- current user base: single-user
- current operational scope: personal media service

## Current Limitations

- media workflow is still relatively manual
- surrounding media automation remains limited
- playback behavior when `vault` is unavailable has not been tested
- hardware transcoding has not been re-verified since media moved to the NAS

## Planned Improvements

- verify hardware transcoding still works correctly against NAS-backed media
- test and document behavior when the NAS is unreachable
- continue refining the media workflow
- record the compose stack under `configs/`
- evaluate whether additional service tooling is actually worth adding