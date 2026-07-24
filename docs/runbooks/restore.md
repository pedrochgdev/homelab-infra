# Restore

## Summary

This document defines how recovery is approached in the homelab, organized around
the tiers defined in [`docs/runbooks/backups.md`](backups.md).

**Restore capability is currently zero.** Not limited — absent. No backups exist,
so none of the procedures below can be executed today. They describe what
recovery will look like once the backup implementation is in place, and they are
written now so that the procedures exist before the incident does.

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

These become executable once the corresponding backup tier exists.

### Tier 1 — personal data and edge configuration

Restores come from the `restic` repository. Prefer the local repository on
`/srv/backup` for speed, and fall back to Backblaze B2 when the local copy is
gone or suspect.

```sh
# What snapshots exist
restic -r /srv/backup/restic snapshots

# Recover a single path from the most recent snapshot
restic -r /srv/backup/restic restore latest \
  --target /tmp/restore --include /srv/nas/users/drocho/<path>

# Same operation against the off-site copy
restic -r b2:<bucket>:/ restore latest --target /tmp/restore --include <path>
```

Restore to `/tmp/restore` first and copy into place after inspection. Restoring
directly over live data turns a recoverable mistake into a second incident.

For `rpi-01` configuration, restore the relevant directory, compare against what
is running, and reload the service rather than replacing files blindly. WireGuard
and Cloudflare credentials are inside these paths, so restored files must keep
restrictive ownership and permissions.

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

### Tier 3 — media

There is no restore path. Media is re-acquired. Use the manifest described in
the backups runbook to determine what was present.

## Recovery Time Expectations

Rough targets, to be validated by the first restore test:

| Scenario | Expected effort |
|---|---|
| Single file from local repository | Minutes |
| Single file from B2 | Minutes, bounded by download speed |
| Full tier 1 restore from B2 | Hours, dominated by 39 GB download |
| One VM from local dump | Under an hour |
| `vault` host rebuild plus array reassembly | A day, largely manual |

## Current Limitations

The homelab does not yet have:

- any backups at all, so no restore is currently possible
- restore-tested backup sets
- documented service rebuild procedures for `atlas` and `virt`
- alerting that would reveal a backup job that stopped running

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

1. implement the backup tiers, following
   [`docs/runbooks/backups.md`](backups.md#implementation-order)
2. perform one small restore test and record the date in the backups runbook
3. create a rebuild checklist for `atlas`
4. create a rebuild checklist for `virt`
5. measure the actual recovery times and replace the estimates above

## Notes

The procedures here are written against a backup implementation that does not
exist yet. They are deliberately specific anyway: writing them now surfaces what
the backup design has to support, and an incident is the worst moment to be
designing a recovery process from scratch.