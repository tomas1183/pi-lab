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