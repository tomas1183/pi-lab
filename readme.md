# Raspberry Pi Homelab - Pi-hole + Unbound

## Overview
This project documents the setup of a Raspberry Pi 5, starting from a
containerized Pi-hole and Unbound DNS stack and growing into a broader
homelab: network segmentation with a zone-based firewall, a reverse proxy,
automated backups, and ~20 containerized services in total.

## Features
- NVMe boot configuration
- SSH key-based authentication
- Docker + Portainer container platform (~20 services)
- Pi-hole DNS filtering with DHCP, Unbound recursive DNS resolver
- VLAN segmentation with a zone-based firewall (UniFi)
- Reverse proxy with friendly internal hostnames (nginx-proxy-manager)
- Automated, encrypted, multi-destination backups (Duplicati)
- Secrets management and public-exposure review
- Monitoring and alerting (Uptime Kuma, Glances)

## Documentation

**Hardware & OS**
- 01-hardware.md
- 02-nvme.md

**Access & Security**
- 03-ssh.md
- 09-secrets-hygiene.md

**Networking**
- 04-network.md
- 08-vlan-segmentation.md
- 10-reverse-proxy-fqdns.md
- 11-wan-troubleshooting.md
- 12-wifi-rf-troubleshooting.md

**Platform & Services**
- 05-docker.md
- 06-dns.md

**Backups**
- 07-backups.md

**Monitoring**
- 13-monitoring.md