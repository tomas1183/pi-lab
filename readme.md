# Raspberry Pi Homelab - Pi-hole + Unbound

## Overview
This project documents the setup of a Raspberry Pi 5, starting from a
containerized Pi-hole and Unbound DNS stack and growing into a broader
homelab: network segmentation with a zone-based firewall, a reverse proxy,
automated backups, a dedicated Windows Server identity lab, and ~20
containerized services in total.

## About / Background
This homelab doubles as hands-on practice tied to an ongoing IT
certification path. Currently held: **CompTIA A+, Network+, Security+,
Cloud+**, **LPI Linux Essentials**, **AWS Certified Cloud Practitioner**,
**ITIL 4 Foundations**, **Microsoft AZ-900 (Azure Fundamentals)**, and a
**BSIT degree from Western Governors University**. Currently in progress:
**AZ-104 (Azure Administrator)**, with **SC-300 (Identity and Access
Administrator)** planned next — the dedicated Windows Server/Active
Directory identity lab documented here exists specifically to get real
hands-on practice for that cert path, not just pass an exam from theory
alone.

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
- Dedicated Windows Server identity lab (AD DS, VLAN-isolated)

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

**Identity Lab**
- 14-identity-lab-addds.md