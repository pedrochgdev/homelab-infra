# atlas

## Summary

`atlas` is the current media server node in the homelab. It runs Jellyfin on Ubuntu Server through Docker and currently stores its media library locally.

Its main architectural change still pending is the migration of media storage from local disk to the NAS on `vault`.

## Role

- Jellyfin media server
- media playback and transcoding node

## Operating System

- Platform: Ubuntu Server
- Version: `24.04`

## Hardware

- CPU: Intel Core i5, approximately 7th generation
- Memory: 8 GB RAM
- GPU: NVIDIA GeForce GTX 1060
- Local Storage: 500 GB

## Service Stack

- Docker version: `29.1.2`
- Primary application: Jellyfin
- Deployment style: Docker-based

## Storage Layout

Current local paths:

- Media: `/srv/media`
- Jellyfin config: `/srv/jellyfin/config`
- Jellyfin cache: `/srv/jellyfin/cache`
- Compose file: `/srv/jellyfin/docker-compose.yml`

At present, media remains local to `atlas`.

## Jellyfin Behavior

- Default port configuration
- Hardware acceleration enabled
- GPU-assisted transcoding in use
- Single-user consumption pattern at present

## Access Model

Jellyfin is currently intended for internal LAN access.

There is no direct public exposure. When remote access is needed, it is reached indirectly through the personal workstation using Tailscale and remote desktop access into the LAN environment.

## Current State

The service is active and functional.

Main architectural limitation:
- media storage is still local rather than centralized on `vault`

## Planned Changes

The main next step for `atlas` is:

- migrate media storage from local disk to the NAS

Planned transport for that migration is currently expected to be SMB, although this may change later if a better option is selected.

## Operational Notes

- Jellyfin is currently the only service running on this node
- content acquisition and media workflow are still relatively manual
- the host may be renamed later to better reflect its service role
- the service works, but the surrounding media workflow is still expected to evolve