# Backups

## Summary

This document defines the backup strategy for the homelab. It is written to be
executed without making further decisions: what is protected, where it goes, how
often, how long it is kept, and how it is verified.

**Current reality, as of the July 2026 audit: there are no backups.** Not
incomplete ones — none.

```
/var/lib/vz/dump/                     empty
/etc/pve/jobs.cfg                     does not exist (no scheduled backup jobs)
qm listsnapshot 100/101/102           no snapshots on any VM
/srv/nas/backups/configs              empty
/srv/nas/backups/exports              empty
/srv/nas/backups/pcs                  empty
/srv/nas/backups/servers/maincore/    two tarballs dated 2026-03-14
```

The only copies that exist are two manual tarballs from four months ago. The
directory structure under `/srv/nas/backups` was created but never filled.

## RAID Is Not a Backup

RAID 1 on `vault` protects against:

- failure of a single data disk

It does not protect against:

- accidental deletion
- file corruption
- ransomware reaching the SMB shares
- permission mistakes
- operating system failure
- array misconfiguration
- host-level failure
- theft, fire, or physical damage

Both mirror members live in the same chassis, in the same room, on the same
power supply. Any strategy must assume the live array can be lost as a unit.

## Tiers

Data is protected according to how hard it would be to recreate, not according to
how much of it there is.

| Tier | Contents | Size | Destination | Frequency | Retention |
|---|---|---|---|---|---|
| 1 — Irreplaceable | `/srv/nas/users`, plus service configuration on `rpi-01` | ~39 GB | Backblaze B2 **and** local LV on `vault` | Daily | 7 daily, 4 weekly, 6 monthly |
| 2 — Rebuildable at cost | The three VMs on `virt` | ~40 GB compressed | Local LV on `vault` | Weekly | 4 |
| 3 — Recreatable | `/srv/nas/content/media` | 769 GB | None, by explicit decision | — | — |

### Tier 1 detail

Two distinct things, both irreplaceable for different reasons:

**Personal data.** `/srv/nas/users`, currently about 39 GB. This is the only
copy that exists anywhere.

**Service configuration on `rpi-01`.** The edge node's configuration exists
solely on its NVMe drive, and reconstructing it by hand would be slow and
error-prone:

- `/etc/nginx/` — reverse proxy definitions for every internal hostname
- `/etc/wireguard/` — VPN server configuration and peer keys
- `/etc/cloudflared/` — tunnel configuration and credentials
- AdGuard Home working directory — filter lists, rewrites, and query settings

Note that `/etc/wireguard/` and `/etc/cloudflared/` contain secrets. This is
precisely why the remote repository must be encrypted client-side.

Also worth including: `/var/www/display/index.html` on `monitor`, which is
authored work existing in exactly one place. See
[`docs/services/display.md`](../services/display.md).

### Tier 3 decision

Media is deliberately not backed up. 769 GB of remote storage is not justified
for content that can be re-acquired, and the cost would dominate the entire
backup budget.

The decision is recorded here rather than left implicit, so it is a choice rather
than an oversight.

One cheap mitigation is worth doing anyway: generate a periodic manifest so the
library contents are known even if the files are gone.

```sh
find /srv/nas/content/media -type f -printf '%p\t%s\n' > /srv/backup/media-manifest.txt
```

The manifest is a few megabytes and belongs in tier 1. Knowing what was lost is
most of the work of rebuilding it.

## Tooling

### Tier 1: restic

`restic` is the right tool here for three specific reasons:

- **Client-side encryption.** The provider stores ciphertext and never holds the
  key. This matters because tier 1 includes WireGuard and Cloudflare credentials.
- **Deduplication and real incrementals.** Only changed blocks are uploaded, so
  daily runs stay cheap after the first.
- **Declarative retention.** `restic forget --prune` enforces the retention
  policy directly, instead of requiring hand-written rotation scripts.

### Tier 2: vzdump

Proxmox's native `vzdump` is already installed and is sufficient for three VMs
that are not classified as irreplaceable. Weekly full dumps with four kept
copies is proportionate.

Proxmox Backup Server is the natural evolution if VM backups later need
deduplication and incrementals, and `vault` has the capacity to host it. It is
not warranted yet.

## Destinations

### Off-site: Backblaze B2

This is the only destination that survives fire, theft, or ransomware, and it is
the one that matters most. Roughly 39 GB at B2 pricing costs on the order of
0.25 USD per month.

Without this tier, everything else is still in the same room.

### Local: new logical volume on `vault`

`vault` has approximately **780 GB of unallocated space** in `vg0`. The `sda4`
partition is 928.5 GiB and only 148 are assigned:

| Volume | Size | Mount |
|---|---|---|
| `vg0-lv_root` | 100 GB | `/` |
| `vg0-lv_var` | 40 GB | `/var` |
| `vg0-lv_swap` | 8 GB | swap |

Creating an `lv_backup` volume there costs nothing and gives fast local restores
without pulling from the internet.

Mount it at `/srv/backup`, deliberately outside `/srv/nas`, so that it is never
reachable through a Samba share. A backup target writable by the same credentials
that ransomware would use is not a backup target.

**This local copy is not an independent copy.** It sits on the same physical
disk as the operating system, on the same host, in the same room. It protects
against accidental deletion and nothing else. Loss of `sda` takes the OS and this
copy together.

## Verification

A backup that has never been restored is an assumption, not a backup.

| Check | Frequency |
|---|---|
| `restic check` against the remote repository | Monthly |
| Restore a known file from tier 1 and compare it | Quarterly |
| Restore one VM to a scratch VMID and boot it | Quarterly |

Record the date of the last successful restore test here, so that a stale test is
visible rather than forgotten:

- Last tier 1 restore test: *never*
- Last tier 2 restore test: *never*

## Monitoring Gap

Backup failure detection currently has nowhere to go. Prometheus scrapes metrics
but there is no `alertmanager` installed and no `rule_files` configured, so a
backup job that silently stops running would not surface until it was needed.

The intended rule, once the alerting layer exists:

> alert when the most recent successful backup is older than 48 hours

This is blocked on deploying Alertmanager, which is tracked in
[`docs/roadmap.md`](../roadmap.md).

Until then, backup success must be checked by hand, and that expectation should
be treated as a known weakness rather than a working process.

## Implementation Order

1. Create `lv_backup` on `vault` and mount it at `/srv/backup`
2. Configure `restic` for tier 1 against B2, plus a local repository on `/srv/backup`
3. Run the first full backup and verify it with `restic check`
4. Schedule the daily run with a systemd timer
5. Configure a weekly `vzdump` job on `virt` writing to `vault`
6. Perform the first restore test and record the date above
7. Add the staleness alert once Alertmanager exists

Steps 1 through 3 close the largest gap. The rest is refinement.

## Notes

This document describes the intended implementation. None of it is in place yet.
It stops being a plan and becomes a runbook at step 6, when the first restore is
proven to work.
