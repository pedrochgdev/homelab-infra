# Jellyfin

## Summary

Jellyfin is the current media service in the homelab and runs on `atlas` using Docker. The service is operational and uses hardware acceleration for transcoding.

The main architectural improvement still pending is the migration of media storage from local disk on `atlas` to centralized storage on `vault`.

## Host

- Node: `atlas`
- Platform: Ubuntu Server `24.04`
- Runtime: Docker `29.1.2`

## Purpose

- host personal media library
- provide media playback and streaming
- support hardware-accelerated transcoding

## Current Paths

- Media: `/srv/media`
- Config: `/srv/jellyfin/config`
- Cache: `/srv/jellyfin/cache`
- Compose file: `/srv/jellyfin/docker-compose.yml`

## Networking

- Port: 8096
- Exposure: internal LAN only

Remote access is currently indirect through the personal workstation using Tailscale and remote desktop access into the LAN environment.

## Media and Storage

Current state:
- media is stored locally on `atlas`

Planned change:
- media should be moved to `vault`
- `atlas` should consume centralized storage instead of relying on local disk

The expected first approach is SMB-based mounting, although this may be revisited later.

## Transcoding

- hardware acceleration enabled
- GPU-assisted transcoding in use

## Usage Pattern

- current user base: single-user
- current operational scope: personal media service

## Current Limitations

- media workflow is still relatively manual
- local media storage creates unnecessary coupling between application host and storage
- surrounding media automation remains limited

## Planned Improvements

- move media library to `vault`
- improve storage separation
- continue refining the media workflow
- evaluate whether additional service tooling is actually worth adding