# Backups

## Summary

This document defines the backup strategy for the homelab. It is written to be
executed without making further decisions: what is protected, where it goes, how
often, how long it is kept, and how it is verified.

## Implementation Status

The July 2026 audit found no backups at all: no scheduled Proxmox jobs, no VM
snapshots, empty directories under `/srv/nas/backups`, and two manual tarballs
from four months earlier.

All three tiers are now implemented.

| Component | Status |
|---|---|
| `lv_backup` on `vg0`, 400 GB, mounted at `/srv/backup` | Done |
| restic repository, local | Done |
| restic repository, Backblaze B2 (`homelab-vault-arpa`) | Done |
| Daily backup of `/srv/nas/users` and `/srv/nas/backups` to both | Done, 03:30 |
| Vaultwarden backup from `atlas` into the staging area | Done, 02:00 |
| `rpi-01` service configuration into the staging area | Done, 02:20 |
| Tier 2, weekly `vzdump` of the three VMs over NFS | Done, Saturdays 01:00 |
| Restore test | **Not done** |

The remaining gap is the one that matters most. Every mechanism above is
unproven until a restore has actually been performed.

### Schedule

```
01:00 Sat  virt    vzdump of all three VMs  → NFS to vault-dump
02:00      atlas   Vaultwarden dump         → staging
02:20      rpi-01  edge configuration       → staging
03:30      vault   restic: local + B2       ← sweeps staging
```

The ordering is deliberate: producers write to the staging area before the
nightly restic run collects it, so a day's dumps land in that same snapshot.

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
| 1 — Irreplaceable | `/srv/nas/users`, `/srv/nas/backups`, plus service configuration on `rpi-01` | ~48 GB | Backblaze B2 **and** local LV on `vault` | Daily | 7 daily, 4 weekly, 6 monthly |
| 2 — Rebuildable at cost | The three VMs on `virt` | ~40 GB compressed | Local LV on `vault` | Weekly | 4 |
| 3 — Recreatable | `/srv/nas/content/media` | 769 GB | None, by explicit decision | — | — |

### The staging pattern

Nodes other than `vault` cannot write into the restic repository, by design:
`/srv/backup` is `root:root` mode `700` and is not shared over the network at
all. If a compromised host or ransomware reached the Samba shares, the encrypted
repository would still be out of reach.

Instead, other nodes drop their dumps into `/srv/nas/backups/<host>/`, which is
on the RAID array and reachable, and the daily restic run on `vault` sweeps that
directory into the protected repository. The staging area is expendable; the
repository is not.

This is what the previously empty `configs`, `exports`, `pcs` and `servers`
directories were for.

Current producers:

| Host | Content | Time |
|---|---|---|
| `atlas` | Vaultwarden database and RSA signing key | 02:00 |
| `rpi-01` | nginx, WireGuard, cloudflared, AdGuard, plus a service inventory | 02:20 |
| `vault` | Sweeps the staging area into both repositories | 03:30 |

Each producer pushes over SSH with a dedicated key authorized on `vault` with
port forwarding, agent forwarding, X11 forwarding and pty allocation disabled,
so those keys can move files and nothing else.

The `rpi-01` tarball contains WireGuard private keys and Cloudflare tunnel
credentials. It is written mode 600 and lands in a repository that is encrypted
client-side, which is why the off-site copy could be entrusted to a third party
at all.

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

Proxmox's native `vzdump` is sufficient for three VMs that are not classified as
irreplaceable. Weekly snapshot-mode dumps with `zstd` compression, retaining
four.

`vault` exports `/srv/backup/dump` over NFS to `virt` and only to `virt`, added
in Proxmox as the `vault-dump` storage. **Only that subdirectory is exported**:
`/srv/backup/restic` stays outside the export, so a compromise of `virt` could
write dumps but could not reach the encrypted repositories.

NFSv4.2 is used, which needs only port 2049. Port 111 is also permitted from
`virt` so that Proxmox's storage probe works.

Proxmox Backup Server is the natural evolution if VM backups later need
deduplication and incrementals, and `vault` has the capacity to host it. It is
not warranted yet.

## Destinations

### Off-site: Backblaze B2

Bucket `homelab-vault-arpa`, private, with an application key scoped to that
bucket alone. Credentials live in `/root/.config/restic/b2.env` on `vault`,
mode 600.

This is the only destination that survives fire, theft, or ransomware. Roughly
48 GB at B2 pricing costs on the order of 0.30 USD per month.

Both repositories use the same key, so there is one secret to protect rather
than two. Lifecycle rules on the bucket are left at *keep all versions*:
retention is enforced by `restic forget`, and a second policy on the provider
side would fight with it.

Note that the restic invocation needs `HOME` defined to use its local cache.
systemd units start without it, so `homelab-backup.service` sets
`Environment=HOME=/root` explicitly. Without that, every run re-downloads index
metadata from B2.

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

## Operational Reference

On `vault`:

| Path | Purpose |
|---|---|
| `/srv/backup/restic` | The repository |
| `/root/.config/restic/password` | Repository key, mode 600 |
| `/usr/local/bin/homelab-backup.sh` | Daily job: backup, then `forget --prune` |
| `homelab-backup.timer` | 03:30, `Persistent=true` |

On `atlas`:

| Path | Purpose |
|---|---|
| `/usr/local/bin/vaultwarden-backup.sh` | `sqlite3 .backup`, tar, push to staging |
| `vaultwarden-backup.timer` | 02:00 |
| `/root/.ssh/id_ed25519` | Push key, authorized on `vault` with forwarding and pty disabled |

**The repository key is the single point of failure for the whole scheme.** It
lives on `vault`, which is the machine being protected. A copy must exist outside
the homelab — in a password manager or on paper — or an off-site restore is
impossible by construction.

## Remaining Work

1. Configure the B2 repository and run the first off-site backup
2. Back up `rpi-01` service configuration through the staging area
3. Schedule the weekly `vzdump` job on `virt`
4. Perform the first restore test and record the date above
5. Add the staleness alert once Alertmanager exists

## Notes

Tier 1 protects against accidental deletion today, and nothing more. Fire, theft
and ransomware are still unmitigated until step 1 is done, and nothing here is
proven until step 4.

