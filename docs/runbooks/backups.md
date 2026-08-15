# Backups

## Summary

This document defines the backup strategy for the homelab. It is written to be
executed without making further decisions: what is protected, where it goes, how
often, how long it is kept, and how it is verified.

## Implementation Status

The July 2026 audit found no backups at all: no scheduled Proxmox jobs, no VM
snapshots, empty directories under `/srv/nas/backups`, and two manual tarballs
from four months earlier.

All three tiers are built. One of them has never worked.

| Component | Status |
|---|---|
| `lv_backup` on `vg0`, 400 GB, mounted at `/srv/backup` | Working |
| restic repository, local | Working |
| restic repository, external disk (`homelab-offsite`) | Working, first copy 2026-08-13 |
| restic repository, Backblaze B2 (`homelab-vault-arpa`) | Set aside; the nightly job still fails on this leg |
| Daily backup of `/srv/nas/users` and `/srv/nas/backups`, local leg | Working, 03:30 |
| Daily backup, B2 leg | Removed 2026-08-13; the nightly job exits clean again |
| Alerting on backup failure and staleness | Done, via `ntfy` |
| Off-site copy on plugging the drive in | Automatic via `udev` |
| Staleness alert if the drive is not seen for 30 days | Done |
| Vaultwarden backup from `atlas` into the staging area | Working, 02:00 |
| `rpi-01` service configuration into the staging area | Working, 02:20 |
| Tier 2, weekly `vzdump` of the three VMs over NFS | Working, Saturdays 01:00 |
| Tier 1 restore test, local repository | Passed 2026-08-13 |
| Tier 1 restore test, off-site repository | Blocked on B2 |
| Tier 2 restore test | Not done |

## Incident record

Two failures found on 2026-08-13, both invisible until someone looked. They are
recorded here rather than quietly fixed, because between them they are the
strongest argument this document can make for the alerting layer it still lacks.

### The off-site copy never existed

Every B2 upload since the repository was created was rejected:

```
403: Cannot upload files, storage cap exceeded.
```

The Backblaze account's storage cap sits below what the repository needs — the
free allowance is 10 GB and tier 1 is roughly 43 GB stored. The daily job logged
`FALLO backup en b2` every single night and the service exited non-zero every
single night, and nothing was watching.

The local leg succeeded throughout, which is why the failure was easy to miss:
snapshots kept appearing on schedule and looked like a working system. **The tier
that protects against fire, theft and ransomware had never held a byte.**

Resolved 2026-08-13 by replacing the destination rather than paying for it. See
[Off-site: external disk](#off-site-external-disk).

Worth recording what the cost actually was, since the decision was made on
price: 43 GB at B2's rate, minus the 10 GB free allowance, comes to about **0.22
USD per month**. The obstacle was putting a card on file, not the amount.

### `vault` was dead for six days

The journal stops mid-routine at 13:10 on 2026-08-05 and resumes at 02:59 on
2026-08-11. No kernel panic, no disk error, no thermal event, no clean shutdown
record — the machine simply stopped. No other node was affected, so it was not a
household power cut.

SMART cleared the disks: zero reallocated sectors, zero pending, zero CRC errors,
all three `PASSED`, and the RAID resynced clean. The board's BIOS dates from
November 2012 and the RAM is non-ECC. An abrupt halt leaving no trace, with
healthy disks and no machine-check events, points at power — an aged supply or a
mains interruption.

Consequences, none of them detected at the time:

- no tier 1 backup ran between 2026-08-05 and 2026-08-11
- the weekly `vzdump` of 2026-08-08 failed with `storage 'vault-dump' is not
  online`, because the NFS export lives on the machine that was down
- the machine did not come back by itself; it stayed off until someone noticed

Two cheap mitigations, in order of value:

1. **A UPS.** It prevents the outage and, more importantly, allows a clean
   shutdown instead of another abrupt one. On the machine holding every backup it
   is the best-value purchase in the homelab.
2. **BIOS "After Power Failure" set to Power On.** That alone would have turned
   six days of outage into two minutes.

Note the shape of the dependency this exposed: `vault` holds the backups *and*
hosts the NFS export that tier 2 writes into. When it goes, both tiers stop
together.

Live verification on 2026-08-15, after recovery, confirmed every producer back
on schedule: the Vaultwarden job on `atlas` and the configuration job on
`rpi-01` both succeeded that morning, and `vault-dump` holds dumps of all three
VMs from the 01:00 run the same day.

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
| 2 — Rebuildable at cost | The three VMs on `virt` | ~33 GB per run, ~109 GB retained | Local LV on `vault` | Weekly | 4 |
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
| `rpi-01` | nginx, WireGuard, cloudflared, AdGuard, the internal CA, plus a service inventory | 02:20 |
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
- `/etc/homelab-ca/` — the internal certificate authority
- AdGuard Home working directory — filter lists, rewrites, and query settings

The CA belongs here for a reason that differs from the rest. Losing it destroys
no data, but it forces creating a new authority and reinstalling the root
certificate by hand on every phone, laptop and workstation that trusts it. It is
irreplaceable in effort rather than in content.

Note that `/etc/wireguard/`, `/etc/cloudflared/` and `/etc/homelab-ca/` all
contain private keys. This is precisely why the remote repository must be
encrypted client-side.

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

### Off-site: external disk

A 1 TB USB drive holding a third restic repository, carried out of the house.

| | |
|---|---|
| Device | Toshiba MQ04UBF100, serial `Z2CPP1F2T` — a 2.5" mechanical drive, not an SSD |
| Filesystem | ext4, label `homelab-offsite`, mounted at `/srv/offsite` |
| Repository | `/srv/offsite/restic`, same key as the other two |
| First copy | 2026-08-13, 20,662 files, 42.8 GiB stored, verified |

**The value of this tier depends entirely on where the drive lives.** Plugged
into `vault` it is a third copy in the same room, on the same power supply,
protecting against nothing the RAID does not already cover. Kept elsewhere and
brought back periodically, it is the only thing standing between a house fire and
total loss.

That distinction cannot be enforced by any script, which is why it is stated here
rather than assumed.

Scripts do enforce the rest:

- a `udev` rule starts `homelab-offsite.service` when a filesystem labelled
  `homelab-offsite` appears, so the procedure is plug in, wait for the phone,
  unplug — a monthly ritual that has to be remembered is a ritual that stops
  happening
- the run backs up, prunes to 12 recent plus 12 monthly, verifies with
  `restic check --read-data-subset=2%`, and unmounts before notifying, so the
  drive is never pulled mid-write
- **on failure the timestamp is not updated and the alert is urgent.** The
  failure mode being designed against is walking away with a drive believed to
  hold a fresh copy
- `homelab-offsite-vigilar.timer` checks daily and warns after 30 days, urgently
  after 60

**`udev` fires more than once for the same disk.** The first real test produced
two runs, at 07:50 and 07:53. They happened to be harmless because the first had
finished, but overlapping runs would have had two `restic` processes contending
for one repository — one would fail, and the failure alert would have fired with
nothing actually wrong. A false alarm is worse than no alarm, because it teaches
you to ignore the real ones.

Two guards now make that impossible rather than unlikely: an `flock` so only one
run can hold the repository, and a check that skips entirely if a copy succeeded
in the last ten minutes.

The drive is mechanical and travels, so it deserves more care than a flash
device: never unplug without unmounting, and do not move it while it is spinning.

**The repository password now matters more than it ever has.** It lives on
`vault`, which is the machine this tier exists to survive. If a fire takes
`vault`, what remains is a drive full of ciphertext. A paper copy must exist, and
must not travel with the drive — the two together defeat the encryption that
makes carrying it safe at all.

### Off-site alternative not taken: Backblaze B2

The bucket `homelab-vault-arpa` and its scoped key still exist, and
`homelab-backup.sh` still tries to use them, so the nightly job will keep
reporting a failed B2 leg until that is removed.

It was set aside on cost preference rather than on the merits. For the record,
43 GB stored at B2's rate less the 10 GB free allowance is about **0.22 USD per
month**, and restoring it once falls inside the free egress allowance. The
obstacle was putting a card on file.

Compared with the external drive it wins on one axis and loses on another: it
needs no discipline and updates nightly without anyone touching anything, but it
costs money and puts the data with a third party. The drive is free and fully
under control, and is exactly as current as the last time someone remembered to
bring it home.

Note for whenever it is reinstated: the restic invocation needs `HOME` defined to
use its local cache. systemd units start without it, so `homelab-backup.service`
sets `Environment=HOME=/root`. Without that, every run re-downloads index
metadata.

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

- Last tier 1 restore test, local repository: **2026-08-13, passed**
- Last tier 1 restore test, off-site repository: *never* — blocked on the B2 cap
- Last tier 2 restore test: *never*

The 2026-08-13 run verified the repository structure with `restic check`,
re-read 2% of the data packs byte for byte, and restored a 12.7 MiB file whose
SHA-256 matched the original exactly. It also confirmed the `rpi-01`
configuration tarball extracts and contains `nginx`, `wireguard`, `cloudflared`
and the AdGuard working directory.

**This is a partial result and should not be read as more.** It proves that
restic, the repository format and the stored password all work. It says nothing
about whether a usable copy exists anywhere other than the machine being backed
up — and that machine went down hard eight days earlier.

Note for whoever writes the next test: the paths inside the `rpi-01` tarball are
relative to a staging directory, so its contents appear as `./nginx` and not
`./etc/nginx`. A verification that greps for the latter reports everything
missing from a perfectly good archive.

## Monitoring Gap

This was not a theoretical gap. It failed twice in one week, and both incidents
above went unnoticed for days precisely because nothing was watching. The
off-site backup failed *every night since it was created* while the system
reported healthy-looking local snapshots.

The information existed the whole time. `homelab-backup.service` exited non-zero
every night and `systemctl` recorded it faithfully. Nothing was reading it.

### Closed, without Alertmanager

Alertmanager is still not installed, and it turned out not to be the blocker.
Once a notification channel existed, the rule this document had been asking for
took about fifteen lines:

| Alert | Trigger | Priority |
|---|---|---|
| Daily backup failed | Non-zero exit from the nightly run | urgent |
| **No successful backup in 48 h** | `homelab-backup-vigilar.timer`, daily at 09:00 | urgent |
| Off-site copy failed | Non-zero exit when the drive is plugged in | urgent |
| Off-site copy stale | 30 days, urgent at 60 | high |
| VPN address changed or failed to publish | `homelab-ddns.sh` | default / urgent |

The 48-hour rule matters for a reason distinct from failure detection, and it is
the one that would have caught August. **A job that stops running produces no
failures to report.** When `vault` was switched off for six days there was
nothing to exit non-zero, no log line, no error — only an absence. Alerting on
staleness catches the absence; alerting on errors never can.

The distinction is worth carrying into whatever gets monitored next: an error
means something went wrong, silence can mean anything at all.

Removing the B2 leg was part of this rather than mere tidying. While it stayed,
`homelab-backup.service` failed every night, so a genuine failure of the local
copy would have produced exactly the signal that had been ignored for weeks. **An
alarm that always sounds is not an alarm.**

This is not hypothetical. The weekly `vzdump` run of 2026-08-08 failed with
`could not activate storage 'vault-dump': storage 'vault-dump' is not online`
— `vault` or its NFS export was unreachable at 01:00 that Saturday — and the
failure went unnoticed until a manual audit found it a week later. The runs
before and after succeeded, so the gap is a missing week of VM backups, but it
demonstrates exactly the silent-failure mode this section warns about.

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

1. **Take the external drive out of the house.** Until it lives elsewhere there
   is still no off-site copy, only a third one in the same room
2. Write the repository password on paper and keep it away from the drive
3. Fit a UPS to `vault`, and set its BIOS to power on after AC loss
4. Restore a VM to a scratch VMID and boot it, for the tier 2 test
5. Review `/srv/nas/backups/servers/maincore`: 9.5 GB of manual tarballs from
   March 2026, a fifth of every copy, probably superseded
6. Deploy Alertmanager for the metrics Prometheus already collects — the backup
   alerts no longer depend on it

## Notes

The single most valuable thing in tier 1 is not configuration or documents. It is
`/srv/nas/users/drocho/media`, 29 GB of personal photographs and video — 68% of
the whole repository, and the only part of this system that cannot be rebuilt,
re-downloaded or reinstalled.

That figure is worth knowing because it was nearly excluded. Media is deliberately
outside the backup at tier 3, and this directory looked like more of the same
until it was examined: 169 files, mostly HEIC and JPEG. The lesson generalises —
a rule written for one directory should not be applied to another by name alone.

With the external drive in place there are now three copies. Two of them sit in
the same room, on the same power supply, in a machine that spent six days of the
last week switched off for reasons still unknown. **The third only counts once it
leaves the house.**

