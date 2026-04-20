# NVMe Setup

## Fresh OS Install (CLI Method)

### Goal
Install Raspberry Pi OS Lite (64-bit) directly to NVMe using CLI tools for lab documentation and learning.

## Step 2 - Download OS

Command:
``` bash
wget https://downloads.raspberrypi.com/raspios_lite_arm64_latest -O raspios.img.xz
```

Result:
Downloading OS image...

## Step 3 - Extract OS

Command:
``` bash
unxz raspios.img.xz
```

Result:
Image extracted successfully to raspios.img

## Step 4 - Write OS to NVMe

Command:
``` bash
sudo dd if=raspios.img of=/dev/nvme0n1 bs=4M status=progress conv=fsync
```

Result:
OS successfully written to NVMe

Notes:
- Current system remained on the SD card during imaging.
- NVMe was prepared from the running SD-based system over SSH.

## Step 5 - Prepare NVMe for first boot

Mounted boot partition:
``` bash
sudo mount /dev/nvme0n1p1 /mnt
```

Enabled SSH:
``` bash
sudo touch /mnt/ssh
```

Unmounted partition:
``` bash
sudo umount /mnt
```

Result:
SSH was enabled for the first boot of the NVMe-based system.

---

## SSH Host Key Warning

Issue:
Received "REMOTE HOST IDENTIFICATION HAS CHANGED" after reinstall.

Cause:
The Raspberry Pi was reinstalled, generating a new SSH fingerprint that did not match the previously stored key on the client machine.

Fix:
ssh-keygen -R 192.168.1.38

Result:
Successfully removed the old key and reconnected after accepting the new host key.

---

## Login after fresh install

Issue:
Unable to SSH into the system after fresh install.

Observed message:
"SSH may not work until a valid user has been set up."

Cause:
New Raspberry Pi OS images do not create a default user and require a user to be configured before allowing SSH access.

Resolution:
Connected to the Raspberry Pi using HDMI and USB keyboard.

A setup wizard was presented on first boot, allowing creation of a user account.

Configured:
- Username: tomas683
- Password: (user-defined)

Result:
Successfully logged in locally and enabled SSH access for the new user.

---

## Notes

- Modern Raspberry Pi OS no longer includes the default "pi" user.
- SSH access requires a user to be created first.
- Using Raspberry Pi Imager with preconfigured user settings can avoid this issue.

---

## Step 6 - Verify NVMe Boot

Command:
``` bash
lsblk
```

Result:
nvme0n1p2 mounted as root (/)
nvme0n1p1 mounted as /boot/firmware

Conclusion:
System successfully booted from NVMe instead of SD card.

Importance:
Confirms the system is no longer running from the SD card and is using NVMe as the primary storage device.

---

## NVMe Setup Complete

Summary:
- Raspberry Pi OS installed via CLI to NVMe
- NVMe configured as primary boot device
- SSH enabled for remote access
- User account created via first-boot wizard
- Successfully logged into system
- Verified system booting from NVMe

Status:
NVMe setup completed successfully. System is ready for further configuration.