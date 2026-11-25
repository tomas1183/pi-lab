## Backup Strategy

### Goal
Ensure Pi-hole configuration and data can be restored in case of system failure or misconfiguration.

---

## Current Primary System: Duplicati (Automated)

The setup below (manual `tar` + `scp`) was the original approach and is kept further down for targeted, pre-change snapshots — but it isn't the main backup system anymore. **Duplicati**, running as its own container (`backup-server`), now handles scheduled, automated backups of the whole Docker host, not just Pi-hole.

**Sources** (2):
- `/home/pi/docker` — every service's bind-mounted config/data (Pi-hole included)
- `/var/lib/docker/volumes` (mounted read-only) — covers the handful of containers still on Docker-managed named volumes rather than bind mounts (see `05-docker.md`'s "Volume Strategy Change" section for why this second source exists at all)

**Destinations** (2):
- Local: `/home/pi/backups` — fast-restore copy, on a **separate 2 TB external USB disk** (ext4, mounted by UUID); see "Moving the local copy to a separate disk" below. It used to live on the same NVMe as the data it protected.
- Offsite: Microsoft OneDrive, via Duplicati's native OneDrive v2 backend — scheduled an hour after the local job to avoid any resource contention

**Schedule**: daily, encrypted (`.zip.aes` archives).

**Gotcha worth noting**: Duplicati's UI has two different-looking "key" fields that are easy to confuse — a `SETTINGS_ENCRYPTION_KEY` (encrypts Duplicati's own internal settings database) and each job's own backup passphrase (encrypts the actual archive files). They are not interchangeable, and the job-edit wizard masks an existing passphrase with no reveal option. If a passphrase is ever needed again, pull it from the job's **⋮ menu → Export → As Command-line** output rather than guessing or resetting it.

---

## Moving the local copy to a separate disk

### Problem
The local backup sat on the same NVMe as the data it protected. That covers accidental deletion and corruption, but a dead NVMe would take the live data and the "fast-restore" copy down together, leaving only the offsite copy.

### What was done
- **Bought a used external 2 TB USB hard drive cheaply and treated it as untrusted until tested.** It was tested before it held any backups.
- **SMART was blocked by the drive's USB adapter.** From the Pi, every `smartctl` passthrough mode failed except the generic SCSI one, which only reports "device lacks SMART capability." Rather than trust a drive with no health data, it was tested on a Windows PC with the vendor's own tool: a short and a full-surface read test, plus the SMART table (reallocated, pending and uncorrectable sector counts all 0; under 64 power-on hours). ATA security was checked too, so the drive wasn't password-locked or encrypted.
- **Formatted ext4 on the Pi, addressing the disk by its stable `/dev/disk/by-id` path** instead of `/dev/sdX`, so a changed device letter could never point the format at the wrong disk.
- **Mounted directly over `/home/pi/backups`** via `/etc/fstab` by UUID, with `nofail` and a 10-second device timeout. Duplicati's existing bind mount needed no stack change, and the Pi still boots if the disk is unplugged.
- **Copied the existing 112 GB of archives with `rsync` and verified before switching:** a second `rsync --dry-run --itemize-changes` listed nothing, meaning every file was identical. Duplicati was stopped during the copy so it couldn't write mid-transfer.
- **Restore test:** one small config file was restored to a temporary location and its SHA-256 matched the live file. The limit of this test is noted in the gotcha above: a restore inside the same Duplicati instance reuses saved settings, so it proves the data is readable, not that the passphrase could be recovered from scratch.

### The failure mode this introduced, and the fix
With `nofail`, a missing disk at boot no longer stops the Pi, but Duplicati would then quietly write backups to the internal disk underneath the empty mount point, with no error anywhere. A watchdog now reports to Uptime Kuma (see `13-monitoring.md`) only while the right disk is mounted, read-write and not nearly full.

### Lessons Learned
- **A cheap used disk is only a backup target after it earns it.** Testing came first, and when the Pi's tools couldn't read SMART, the answer was a different tool, not skipping the check.
- **Verify a copy, then switch.** A dry-run `rsync` is a cheap, unambiguous "identical" test before changing what the system points at.
- **`nofail` trades a loud failure for a silent one.** If boot shouldn't depend on the disk, something else has to notice when it's gone.
- **Bus-powered USB drives and a Pi are a power question.** Plugging in a second bus-powered drive alongside the first coincided with a power fault and a hard reboot, so it stayed off the Pi.

---

## Manual Pi-hole Backup (Secondary, Targeted Use)

Still useful for a quick, deliberate snapshot immediately before a risky Pi-hole change — faster to reason about than digging through a multi-source automated backup when all that's needed is "give me back exactly what Pi-hole looked like five minutes ago."

## What is Backed Up

Directory:
~/containers/pihole/

Includes (but not limited to):
- Pi-hole configuration
- Gravity database (blocklists)
- DNS settings
- Custom configurations

---

## Manual Backup

Command:
``` bash
sudo tar -czvf pihole-backup-$(date +%F).tar.gz ~/containers/pihole
```

Result:
Creates a compressed backup archive in the current directory.

---

## Backup Location

Default:
~/ (home directory of the Raspberry Pi)

Optional:
Backups can be moved to:
- External storage
- Another system
- Cloud storage

---

## Restore Procedure

Warning:
Ensure correct backup file is selected before extraction to avoid overwriting valid configurations.

Steps:

1. Stop containers:
``` bash
docker stop pihole unbound || true
```

2. (Optional but recommended) Remove existing configuration:
``` bash
sudo rm -rf ~/containers/pihole
```

3. Extract backup:
``` bash
sudo tar -xzvf pihole-backup-YYYY-MM-DD.tar.gz -C ~
```

4. Redeploy Docker stack via Portainer

5. Verify DNS functionality

---

## Notes

- Backup should be performed before any updates or major changes
- Backups are lightweight and quick to create
- Keeping multiple backups is recommended
- Use date-based filenames to maintain version history of backups
- Periodically test restore procedure to ensure backups are valid

---

### Permissions Note

Backup must be executed with sudo to ensure all container-owned files are included.

---

### Off-Device Backup

Command (Windows PowerShell):
scp tomas683@192.168.1.38:~/pihole-backup-YYYY-MM-DD.tar.gz .

Result:
Backup copied from Raspberry Pi to local system and stored in a synchronized OneDrive directory.