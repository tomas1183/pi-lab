## Backup Strategy

### Goal
Ensure Pi-hole configuration and data can be restored in case of system failure or misconfiguration.

---

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
sudo tar -czvf pihole-backup-$(date +%F).tar.gz ~/containers/pihole

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
docker stop pihole unbound || true

2. (Optional but recommended) Remove existing configuration:
sudo rm -rf ~/containers/pihole

3. Extract backup:
sudo tar -xzvf pihole-backup-YYYY-MM-DD.tar.gz -C ~

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