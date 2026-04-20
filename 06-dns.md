## Pi-hole and Unbound Stack

### Goal
Deploy Pi-hole and Unbound as containerized services to provide network-wide DNS filtering, recursive DNS resolution, and centralized DHCP management.

---

## Platform

- Raspberry Pi 5 (NVMe boot)
- Docker Engine (Host Networking Mode)
- Portainer CE
- Network: Host mode for DHCP discovery

---

## Architecture

DNS flow:
Client → Pi-hole (DHCP/DNS) → Unbound → Internet → Response

- **Pi-hole:** Handles DNS filtering, blocklists, and authoritative DHCP.
- **Unbound:** Performs recursive DNS resolution (no third-party upstream).
- **Network Mode:** `host` mode is utilized to allow Pi-hole to listen for DHCP broadcast packets and see individual client MAC addresses.

## Design Decisions

- **Host Mode:** Switched from bridge to host networking to enable DHCP server functionality and per-client visibility.
- **DHCP Authority:** Migrated DHCP from the Orbi router to Pi-hole to allow for Identity-based DNS group assignment.
- **IAM Groups:** Implemented "Least Privilege" DNS by segmenting devices into Computers, IoT, and Infrastructure groups.
- Used Docker for service isolation and portability

---

## Step 1 - Prepare Persistent Storage

Commands:
``` bash
mkdir -p ~/containers/pihole/etc-pihole
mkdir -p ~/containers/pihole/etc-dnsmasq.d
```

Result:
Directories created for persistent Pi-hole configuration and DNS settings.

---

## Step 2 - Deploy Stack in Portainer (Host Mode)

**Note:** The stack was redeployed using `network_mode: host` in the docker-compose configuration to allow the Pi-hole to act as the network's DHCP server.

Stack Name:
pihole

Compose Configuration:

```yaml
services:
  unbound:
    container_name: unbound
    image: mvance/unbound-rpi:latest
    restart: unless-stopped
    networks:
      dns_net:
        ipv4_address: 172.30.0.2

  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    hostname: framboise
    restart: unless-stopped
    network_mode: host
    shm_size: '256mb' # Prevents 'No space left on device' shared memory errors
    depends_on:
      - unbound
    cap_add:
      - NET_ADMIN # Required for DHCP functionality in host mode
    environment:
      TZ: "America/New_York"
      FTLCONF_dns_upstreams: "172.30.0.2#53"
      FTLCONF_webserver_api_password: "REDACTED"
      FTLCONF_webserver_port: 8080
      FTLCONF_dns_listeningMode: "all"
      DNSMASQ_LISTENING: "all" # Forces listening on all interfaces in host mode
    volumes:
      - /home/tomas683/containers/pihole/etc-pihole:/etc/pihole
      - /home/tomas683/containers/pihole/etc-dnsmasq.d:/etc/dnsmasq.d

networks:
  dns_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/24
```

Result:
Pi-hole and Unbound containers are operational with direct access to the host's network interfaces.

---

## Step 3 - Verify Container Status

Command:
``` bash
docker ps
```

Result:
- pihole → running
- unbound → running

---

## Step 4 - DHCP Migration

**Action:**
1. Disabled DHCP server on Netgear Orbi (AP Mode).
2. Enabled DHCP server within Pi-hole settings.
3. Configured static reservations for core infrastructure (.1 - .9) and fixed workstations (.10 - .99).

## Step 5 - DNS Validation (Local)

Command:
dig google.com @192.168.1.38

Observed:
- status: NOERROR
- valid IP returned

Result:
Successful DNS resolution.

Conclusion:
Pi-hole is responding on port 53 and correctly forwarding queries to Unbound.

---

## Step 6 - External Client Test

Command (Windows):
nslookup google.com 192.168.1.38

Result:
Successful DNS resolution from external client.

Conclusion:
Network devices can reach Pi-hole and receive valid DNS responses.

---

## Step 7 - Pi-hole Web Interface

URL:
http://192.168.1.38:8080

Result:
Dashboard accessible and query log updating in real time.

---

**Verification:**
Command:
ipconfig /all

Result:
DHCP Server confirmed as 192.168.1.38.

---

## Step 8 - Blocklist Configuration

Configured lists:

- StevenBlack hosts
- OISD (https://big.oisd.nl)
- AdGuard DNS filter

Action:
Updated gravity database.

Result:
Enhanced ad and tracker blocking across the network.

---

## Step 9 - Identity & Group Management

**Groups Configured:**
- **Computers:** Balanced blocking for workstations (Hyte-Assassin).
- **IoT:** Aggressive telemetry blocking for SmartThings and smart devices.
- **Infrastructure:** No blocking for the Orbi Satellite and Printer to ensure stability.

---

## Final Result

- Pi-hole running in Docker (Host Mode).
- Full per-device visibility in Query Logs.
- Authoritative DHCP managed by Raspberry Pi 5.
- Multi-tier DNS filtering active via IAM Groups.

---

## Lessons Learned

- **Bridge vs. Host:** Bridge networking masks client IPs behind the gateway; Host mode is required for full DHCP/DNS visibility.
- **DHCP Handoff:** Disabling the gateway DHCP before enabling the server DHCP prevents IP conflicts.
- **Logical Segmentation:** Organizing devices into groups significantly simplifies the management of "chatty" IoT devices without breaking functionality for main workstations.