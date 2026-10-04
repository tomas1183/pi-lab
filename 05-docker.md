## Container Platform Setup

### Goal
Install and configure Docker to support containerized services.

---

### Step 1 - Install Docker

Command:
``` bash
curl -fsSL https://get.docker.com | sh
```

Result:
Docker Engine and required components installed successfully.

---

### Step 2 - Configure User Permissions

Command:
``` bash
sudo usermod -aG docker $USER
```

Applied group change:
``` bash
newgrp docker
```

Result:
User can run Docker commands without requiring sudo.

---

### Step 3 - Verify Docker Installation

Command:
``` bash
docker version
```

Result:
Docker client and server both reported successfully.

Conclusion:
Docker Engine is installed and operational.

---

### Step 4 - Test Docker Functionality

Command:
``` bash
docker run hello-world
```

Initial Result:
Permission denied while trying to connect to the Docker socket.

Cause:
The current shell session had not yet picked up the new docker group membership.

Fix:
``` bash
sudo usermod -aG docker $USER
newgrp docker
```

Retest:
``` bash
docker run hello-world
```

Final Result:
Docker successfully pulled and executed the hello-world container.

Conclusion:
Docker is fully functional and can run containers.

---

### Step 5 - Confirm Environment

Command:
``` bash
docker version
```

Result:
Client and server versions displayed correctly.

Conclusion:
Docker environment is stable and ready for container deployment.

---

## Portainer Setup

### Step 6 - Create Persistent Volume

Command:
``` bash
docker volume create portainer_data
```

Result:
Persistent storage created for Portainer configuration.

---

### Step 7 - Deploy Portainer Container

Command:
``` bash
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Result:
Portainer container deployed and running.

---

### Step 8 - Verify Container

Command:
``` bash
docker ps
```

Result:
Portainer container confirmed running.

---

### Step 9 - Access Web Interface

URL:
http://192.168.1.38:9000

Result:
Portainer web interface accessible.
Admin account created and local Docker environment connected.

---

Note:
Docker installation script is a convenience method. For production environments, manual installation via official package repositories is recommended.

---

## Current State (as of October 2026)

What started as a two-container DNS stack has grown into a ~20-container homelab, all managed through Portainer CE. Current inventory by function:

| Category | Containers |
|---|---|
| DNS | pihole, unbound |
| Networking / remote access | tailscale, nginx-proxy-manager, duckdns |
| Security | crowdsec, crowdsec-firewall-bouncer |
| Media | stremio, aiostreams, gluetun (VPN), prowlarr, flaresolverr |
| Home automation | homeassistant |
| Monitoring / platform | portainer, watchtower, glances, uptime-kuma, homepage |
| Backup | backup-server (Duplicati, see `07-backups.md`) |
| AI-assisted administration | claude-code (a persistent Claude Code CLI session, containerized for isolation — no `docker.sock` mount, no host networking; it manages the rest of this stack by SSHing back out to the host like any other remote admin session, not by running with elevated container privileges) |

### Volume Strategy Change: Named Volumes → Bind Mounts

**Original approach** (Step 6 above): Docker-managed named volumes (`portainer_data`, etc.) — Docker's default and the path of least resistance when first standing up a container.

**Problem found later**: the automated backup system (Duplicati, see `07-backups.md`) only backs up `/home/pi/docker` on the host. Docker-managed named volumes live under `/var/lib/docker/volumes/`, a completely separate path — meaning any container left on a named volume was invisible to the backup job. An audit found 8 affected volumes across several containers, including this `claude-code` container's own config/workspace and Portainer's own stack definitions.

**Fix**: migrated the affected containers (Tailscale, CrowdSec, claude-code) from named volumes to explicit bind mounts under `/home/pi/docker/<service>/`, matching the convention already used by Pi-hole and most other services. `portainer_data` was deliberately left on its named volume — Portainer isn't itself a Portainer-managed stack, so migrating it would mean a raw `docker stop`/`recreate` with no "just click Update the stack" fallback if something went wrong while Portainer itself was the thing being taken down. Instead, `/var/lib/docker/volumes` was mounted read-only into the backup container as a second source, covering `portainer_data` (and anything else left on named volumes) without needing to touch it directly.

**Lesson**: a backup job's source path is a claim about coverage, not a guarantee — it's worth periodically checking that every container's actual mount list falls inside that path, especially after adding a new service that defaults to a named volume.