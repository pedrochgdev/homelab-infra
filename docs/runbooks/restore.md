# Restore

## Summary

This document defines how recovery is approached in the homelab, organized around
the tiers defined in [`docs/runbooks/backups.md`](backups.md).

All three backup tiers are implemented, so the procedures below are executable.
The tier 1 restore test passed against the local repository on 2026-08-13,
proving restic, the repository format and the stored password. Tier 2 has never
been exercised, and no restore has been performed from the off-site drive, so
those procedures remain assumptions until tested.

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

## Restore Procedures by Tier

### Tier 1 — personal data and edge configuration

Restores come from the `restic` repositories. Prefer the local repository on
`/srv/backup` for speed, and fall back to the off-site external drive (label
`homelab-offsite`, mounted at `/srv/offsite`) when the local copy is gone or
suspect. The same key opens every repository.

```sh
# What snapshots exist
restic -r /srv/backup/restic snapshots

# Recover a single path from the most recent snapshot
restic -r /srv/backup/restic restore latest \
  --target /tmp/restore --include /srv/nas/users/drocho/<path>

# Same operation against the off-site drive, once plugged in and mounted
restic -r /srv/offsite/restic restore latest --target /tmp/restore --include <path>
```

Restore to `/tmp/restore` first and copy into place after inspection. Restoring
directly over live data turns a recoverable mistake into a second incident.

For `rpi-01` configuration, restore the relevant directory, compare against what
is running, and reload the service rather than replacing files blindly. WireGuard
credentials, Cloudflare credentials and the internal CA's root key are inside
these paths, so restored files must keep restrictive ownership and permissions.

The `rpi-01` tarball stores paths relative to a staging directory, so its
contents are `./nginx`, `./wireguard`, `./cloudflared`, `./homelab-ca` — not
`./etc/nginx`. Expect that when listing it, and do not conclude the archive is
empty because the `etc/` prefix is missing.

### Tier 2 — virtual machines

```sh
# Available dumps
ls /srv/backup/dump/

# Restore to a scratch VMID to verify before touching the original
qmrestore /srv/backup/dump/vzdump-qemu-101-<timestamp>.vma.zst 999

# Restore in place, overwriting the existing VM
qmrestore /srv/backup/dump/vzdump-qemu-101-<timestamp>.vma.zst 101 --force
```

Always restore to a scratch VMID first when the original still exists. `--force`
destroys the current disk.

**Bring the restored VM up with its network interface down.** A restored guest
carries the original's static address, so booting a copy of `dns` alongside the
real one puts two hosts on `192.168.1.52` and takes out the secondary resolver
for the whole network — during what was supposed to be a safe verification.

```sh
qm set 999 --net0 virtio=<mac>,bridge=vmbr0,link_down=1
qm start 999
# verify it reaches a login prompt, then
qm destroy 999
```

Note also that the dumps live on `vault` and are reached over NFS from `virt`.
If `vault` is the thing that failed, tier 2 is unreachable until it is back —
the two are not independent.

### Tier 3 — media

There is no restore path. Media is re-acquired. Use the manifest described in
the backups runbook to determine what was present.

## Recovery Time Expectations

Rough targets, to be validated by the first restore test:

| Scenario | Expected effort |
|---|---|
| Single file from local repository | Minutes |
| Single file from the off-site drive | Minutes, once the drive is fetched and mounted |
| Full tier 1 restore from the off-site drive | Hours, dominated by copying ~43 GB over USB |
| One VM from local dump | Under an hour |
| `vault` host rebuild plus array reassembly | A day, largely manual |

## Current Limitations

The homelab does not yet have:

- a tested tier 2 restore, or any restore from the off-site drive
- documented service rebuild procedures for `atlas` and `virt`

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

1. restore a VM to a scratch VMID and boot it, completing the tier 2 test
2. create a rebuild checklist for `atlas`
3. create a rebuild checklist for `virt`
4. measure the actual recovery times and replace the estimates above

## Notes

These procedures were written before the backup implementation existed, and the
implementation now matches them. What remains is proving them: until the first
restore test is performed, every procedure here is untested on real data, and an
incident is the worst moment to discover a gap.