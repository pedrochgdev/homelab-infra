# Remote Boot

## Summary

The remote boot service makes it possible to power on the workstation from
anywhere and choose which operating system it boots — Arch Linux or Windows —
before it starts, without touching the machine. Control is SSH-only, through
the existing WireGuard tunnel into `rpi-01`.

Two mechanisms combine to make this work:

- **Wake-on-LAN**: `rpi-01` sends a magic packet to the workstation's NIC.
- **PXE boot menu**: the workstation boots over the network first, pulls a
  GRUB menu from `rpi-01`, and chainloads the OS that the menu's default
  points at. The default is a one-line file on the Pi, editable over SSH.

Every failure mode in the chain degrades to booting Arch locally. If `rpi-01`
is down entirely, the workstation boots Arch from its own disk after a PXE
timeout.

## Hosts

| Host | Role |
|---|---|
| `rpi-01` (`192.168.1.30`) | Wake trigger, proxyDHCP, TFTP, boot menu |
| workstation (`192.168.1.109`) | Target machine, Arch + Windows dual boot |

## Usage

From any machine that can reach `rpi-01` (LAN or WireGuard):

```bash
ssh drocho@192.168.1.30

boot-pc arch       # set default to Arch and send the magic packet
boot-pc windows    # set default to Windows and send the magic packet

set-boot status    # show the current default
set-boot arch      # change the default without waking the machine
wake-pc            # send the magic packet without changing the default
```

The menu also appears on the physical monitor for 5 seconds on every boot, so
local use works unchanged: press an arrow key to pick an OS, or let the
default win.

## Architecture

```
[phone/laptop anywhere]
   └─ WireGuard → rpi-01
        └─ ssh → boot-pc {arch|windows}
             ├─ set-boot: writes /srv/tftp/grub/default.cfg
             └─ wake-pc:  magic packet → bc:fc:e7:08:b3:d9

[workstation powers on] → UEFI PXE (first in BootOrder)
   └─ dnsmasq proxyDHCP on rpi-01 → TFTP → grub/netboot.efi
        └─ embedded config fetches (tftp,192.168.1.30)/grub/grub.cfg
             ├─ "Arch Linux (GRUB local)" → chainload ESP AA65-6A02
             │                              /EFI/grub_uefi/grubx64.efi
             ├─ "Windows"                 → chainload ESP 4CF1-D4B2
             │                              /EFI/Microsoft/Boot/bootmgfw.efi
             └─ "Salir al boot local"     → exit to firmware
```

The router keeps handing out IP addresses; dnsmasq runs in proxyDHCP mode and
only supplies the PXE boot information. AdGuard Home keeps port 53 — dnsmasq
runs with `port=0` (no DNS at all).

## Components on rpi-01

### dnsmasq (proxyDHCP + TFTP)

Config: `/etc/dnsmasq.d/pxe-remote-boot.conf`

```
port=0
interface=eth0
bind-interfaces
dhcp-range=192.168.1.0,proxy
dhcp-match=set:efi64,option:client-arch,7
pxe-service=tag:efi64,x86-64_EFI,"Menu remoto",grub/netboot.efi
dhcp-boot=tag:efi64,grub/netboot.efi
enable-tftp
tftp-root=/srv/tftp
tftp-port-range=10000,10100
log-dhcp
```

### TFTP tree

```
/srv/tftp/
├── grubnetx64.efi          # retired: Ubuntu's signed netboot GRUB (see History)
└── grub/                   # owned by drocho → updates need no sudo
    ├── netboot.efi         # custom GRUB with embedded config (the boot file)
    ├── grub.cfg            # the remote menu
    └── default.cfg         # one line: `set default=0|1` (0=Arch, 1=Windows)
```

### Control scripts

- `/usr/local/bin/wake-pc` — `wakeonlan -i 192.168.1.255 bc:fc:e7:08:b3:d9`
- `/usr/local/bin/set-boot {arch|windows|status}` — rewrites `default.cfg`
- `/usr/local/bin/boot-pc {arch|windows}` — `set-boot` + `wake-pc`

All three run as `drocho` without sudo (the `grub/` directory is
drocho-owned).

### Firewall (ufw)

| Rule | Purpose |
|---|---|
| `67/udp on eth0 from any` | PXE DHCP discover — **source is `0.0.0.0`**, so this rule must not filter by source network |
| `69/udp from 192.168.1.0/24` | TFTP requests |
| `4011/udp from 192.168.1.0/24` | PXE proxy boot server port |
| `10000:10100/udp from 192.168.1.0/24` | TFTP data (dnsmasq `tftp-port-range`) |

## The custom netboot.efi

Ubuntu's signed `grubnetx64.efi` failed under proxyDHCP: because the boot
information comes from the proxy offer rather than the main DHCP lease, GRUB
never learned the TFTP server address, could not fetch its `grub.cfg`, and
dropped to a bare `grub>` prompt — leaving the machine stranded until a
physical power cycle.

The replacement is a ~660 KB GRUB image built with `grub-mkstandalone` on the
workstation (Secure Boot is disabled, so it does not need signing). Its
embedded config hard-codes the server address and falls back to the local
disk, so a dead `grub>` prompt cannot happen again:

```
insmod efinet
insmod net
insmod tftp
insmod part_gpt
insmod fat
insmod chain
insmod search_fs_uuid

echo "[netboot] Pidiendo IP por DHCP..."
net_bootp
set net_default_server=192.168.1.30

echo "[netboot] Cargando menu remoto desde 192.168.1.30..."
configfile (tftp,192.168.1.30)/grub/grub.cfg

# Reached only if the remote menu could not be loaded.
echo "[netboot] Menu remoto NO disponible -> arrancando Arch local"
sleep 2
search --fs-uuid --set=root AA65-6A02
chainloader /EFI/grub_uefi/grubx64.efi
boot

echo "[netboot] Fallback local fallo -> devolviendo control al firmware"
sleep 3
exit
```

Rebuild recipe (on the workstation, then copy to the Pi — no sudo needed):

```bash
grub-mkstandalone -O x86_64-efi -o netboot.efi --themes= --fonts= --locales= \
  --install-modules="normal configfile chain tftp net efinet search \
                     search_fs_uuid part_gpt fat echo test true sleep \
                     reboot minicmd ls" \
  "boot/grub/grub.cfg=netboot-embedded.cfg"
scp netboot.efi drocho@192.168.1.30:/srv/tftp/grub/netboot.efi
```

The image was validated in QEMU/OVMF (headless, QMP screendumps) before ever
touching the real machine: both the happy path (menu loads from the Pi) and
the failure path (TFTP unreachable → falls through to the local-disk
fallback → exits cleanly to firmware) were exercised.

## The remote menu

`/srv/tftp/grub/grub.cfg`:

```
set timeout=5
set timeout_style=menu
set fallback="2"

source (tftp,192.168.1.30)/grub/default.cfg

menuentry "Arch Linux (GRUB local)" {
  search --fs-uuid --set=root AA65-6A02
  chainloader /EFI/grub_uefi/grubx64.efi
}

menuentry "Windows" {
  search --fs-uuid --set=root 4CF1-D4B2
  chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}

menuentry "Salir al boot local (firmware)" {
  exit
}
```

`fallback="2"` means a failed chainload jumps to the exit entry, which hands
control back to the firmware, which continues down the BootOrder to the local
GRUB — ending in Arch.

## Workstation configuration

- NIC: Intel I226-V, `eno1`, MAC `bc:fc:e7:08:b3:d9`
- WoL armed: `ethtool -s eno1 wol g`, persisted via NetworkManager
  (`802-3-ethernet.wake-on-lan magic`)
- UEFI: Network Stack enabled (IPv4 + IPv6), Secure Boot disabled
- BootOrder: `0003` (PXE IPv4) → `0002` (local GRUB) → `0000` (Windows).
  The ASUS firmware tends to re-insert `0004` (PXE IPv6) after changes;
  it is harmless but adds ~1 min of timeout when the Pi is down.
- LUKS root: TPM2 auto-unlock enrolled (`systemd-cryptenroll`, PCR 7,
  keyslot 1; the passphrase remains in keyslot 0 as fallback) so remote boots
  do not hang at the passphrase prompt. Weakness accepted: with Secure Boot
  off, whole-machine theft can boot to the unlocked disk.

## Failure modes

| Failure | Result |
|---|---|
| `rpi-01` down / dnsmasq stopped / port blocked | Firmware PXE times out → next BootOrder entry → local GRUB → Arch |
| TFTP dies mid-download of `netboot.efi` | Firmware abandons PXE → local GRUB → Arch |
| GRUB gets no DHCP lease | Embedded config falls through → chainloads local Arch |
| Remote menu unreachable (the old `grub>` case) | Embedded fallback → chainloads local Arch |
| Chosen menu entry fails | `fallback="2"` → exit to firmware → local GRUB → Arch |
| Embedded fallback also fails | `exit` → firmware continues BootOrder → Arch |

Boot cost when everything works: ~52 s total (~25 s firmware including PXE,
~13 s loader, ~14 s kernel to desktop). When the Pi is down, add PXE timeouts
(~1–2 min) before the local fallback.

## History

- The first PXE attempt never reached dnsmasq: the ufw rule for port 67
  filtered by source `192.168.1.0/24`, but PXE DHCP discovers arrive from
  `0.0.0.0`. The Pi's router got the broadcast (no firewall) while dnsmasq
  never saw it — the firmware waited ~2 min 25 s through both PXE entries
  before falling back to local boot.
- The second attempt (rule fixed) delivered Ubuntu's `grubnetx64.efi`, which
  loaded and then stalled at `grub>` for the proxyDHCP reason described
  above. That motivated the custom image with the embedded config.
- `grubnetx64.efi` is kept in `/srv/tftp/` only as an artifact; nothing
  references it.

## Pending

- ~~Physical-presence checks before trusting power-off + WoL end to end~~
  **Done 2026-08-25**: WoL from full power-off verified — `boot-pc` on `rpi-01`
  powered the workstation on, so Power On By PCI-E and ErP are correctly set.
- Windows side: disable Fast Startup or enable "Wake on Magic Packet" in the
  Intel driver, so WoL works after a Windows shutdown.
- Optionally disable IPv6 in the BIOS Network Stack to drop the useless
  `0004` PXE entry and its timeout when the Pi is down.
- DHCP reservation for `192.168.1.109` on the router (noted elsewhere in
  this repo as well).
