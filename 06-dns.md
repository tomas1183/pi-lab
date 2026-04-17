## Pi-hole and Unbound Stack

### Goal
Deploy Pi-hole and Unbound as containerized services to provide network-wide DNS filtering and recursive DNS resolution.

---

## Platform

- Raspberry Pi 5 (NVMe boot)
- Docker Engine
- Portainer CE
- Network: Docker bridge network

---

## Architecture

DNS flow:

Client → Router → Pi-hole → Unbound → Internet → Response

- Pi-hole handles DNS filtering and blocklists
- Unbound performs recursive DNS resolution (no upstream provider)

Note:
Pi-hole communicates with Unbound using Docker internal networking via the service name "unbound".

Note:
Docker provides internal DNS resolution, allowing containers to communicate using service names.

## Design Decisions

- Used Docker for service isolation and portability
- Used Unbound for privacy (no third-party DNS providers)
- Retained router DHCP for simplicity and network stability

---

## Step 1 - Prepare Persistent Storage

Commands:
mkdir -p ~/containers/pihole/etc-pihole
mkdir -p ~/containers/pihole/etc-dnsmasq.d

Result:
Directories created for persistent Pi-hole configuration and DNS settings.

---

## Step 2 - Deploy Stack via Portainer

Stack Name:
pihole

Compose Configuration:

```yaml
services:
  unbound:
    container_name: unbound
    image: mvance/unbound-rpi:latest
    restart: unless-stopped

  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    hostname: framboise
    restart: unless-stopped
    depends_on:
      - unbound
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    environment:
      TZ: "America/New_York"
      FTLCONF_dns_upstreams: "unbound#53"
      FTLCONF_webserver_api_password: "REDACTED"
      FTLCONF_dns_listeningMode: "all"
    volumes:
      - /home/tomas683/containers/pihole/etc-pihole:/etc/pihole
      - /home/tomas683/containers/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
```
Result:
Pi-hole and Unbound containers deployed and running.

---

## Step 3 - Verify Container Status

Command:
docker ps

Result:
- pihole → running
- unbound → running

---

## Step 4 - DNS Validation (Local)

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

## Step 5 - External Client Test

Command (Windows):
nslookup google.com 192.168.1.38

Result:
Successful DNS resolution from external client.

Conclusion:
Network devices can reach Pi-hole and receive valid DNS responses.

---

## Step 6 - Pi-hole Web Interface

URL:
http://192.168.1.38:8080

Result:
Dashboard accessible and query log updating in real time.

---

## Step 7 - Router Integration

Router: Netgear Orbi RBR750

Configuration:
- DHCP: Enabled on router
- DNS Server: Set to 192.168.1.38 (Pi-hole)

Result:
All network clients use Pi-hole for DNS.

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

## Troubleshooting

### Issue 1 - DNS Failure

Problem:
DNS queries failed after modifying upstream configuration.

Cause:
Incorrect upstream used:
127.0.0.1#5335 (invalid for Docker bridge networking)

Fix:
Updated upstream to:
FTLCONF_dns_upstreams: "unbound#53"

Result:
DNS functionality restored.

---

### Issue 2 - DHCP Migration Attempt

Problem:
Enabling Pi-hole DHCP caused network instability and loss of connectivity.

Cause:
- Docker bridge networking limitations
- Router DNS proxy behavior
- DHCP broadcast handling conflicts

Resolution:
- Disabled Pi-hole DHCP
- Re-enabled DHCP on router

Result:
Stable DNS-only deployment achieved.

---

## Client Visibility Limitation

Observation:
All DNS queries appear as originating from the router (192.168.1.1).

Cause:
Orbi router acts as a DNS proxy, forwarding DNS requests instead of allowing direct client communication.

Impact:
- No per-device visibility in Pi-hole logs
- Conditional forwarding not applicable

Decision:
Retained router DHCP for stability and deferred per-device visibility improvements.

---

## Final Result

- Pi-hole running in Docker
- Unbound providing recursive DNS resolution
- Network-wide DNS filtering active
- Router integrated with Pi-hole DNS
- Stable and production-ready configuration

---

## Notes

- Container-to-container communication uses Docker networking (service name resolution)
- Pi-hole upstream correctly points to Unbound container
- Router DHCP retained for simplicity and stability
- DHCP migration may be revisited in the future for per-device visibility

---

## Lessons Learned

- Docker networking model affects how services communicate (bridge vs host)
- Service names can be used for container-to-container DNS resolution
- Router DNS behavior can interfere with visibility and control
- DHCP transitions must be performed carefully to avoid network outages
- Validating each layer (container → DNS → client) is critical before making changes