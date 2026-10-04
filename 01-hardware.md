# Hardware Setup

## Goal
Set up Raspberry Pi 5 with NVMe storage.

## Hardware Used
- Raspberry Pi 5 (8GB RAM)
- Geekworm X1001 NVMe HAT
- Kingston NV3 1TB SSD
- Geekworm P579 V2 Case (active cooling)
- RasTech 5V 5A Power Supply

Runs headless, Ethernet-only — no Wi-Fi, no monitor/keyboard attached day-to-day.

## Steps Taken
- Assembled Pi with NVMe HAT and SSD
- Installed into case
- Powered on system
- Verified NVMe detection

## Verification Command Used
```bash
lsblk
```

## Result
NVMe drive detected successfully as /dev/nvme0n1

## Notes
Everything powered on without issues.