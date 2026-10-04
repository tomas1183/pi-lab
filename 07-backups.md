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
- Local: `/home/pi/backups` — fast-restore copy, same host
- Offsite: Microsoft OneDrive, via Duplicati's native OneDrive v2 backend — scheduled an hour after the local job to avoid any resource contention

**Schedule**: daily, encrypted (`.zip.aes` archives).

**Gotcha worth noting**: Duplicati's UI has two different-looking "key" fields that are easy to confuse — a `SETTINGS_ENCRYPTION_KEY` (encrypts Duplicati's own internal settings database) and each job's own backup passphrase (encrypts the actual archive files). They are not interchangeable, and the job-edit wizard masks an existing passphrase with no reveal option. If a passphrase is ever needed again, pull it from the job's **⋮ menu → Export → As Command-line** output rather than guessing or resetting it.

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