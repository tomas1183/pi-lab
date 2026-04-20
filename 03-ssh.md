## Security Note

Key-based authentication significantly improves security by eliminating password-based login attempts and reducing exposure to brute-force attacks.

---

## SSH Key Authentication Setup

### Goal
Enable secure, passwordless SSH login using key-based authentication.

---

### Step 1 - Generate SSH Key (Client)

Command (Windows PowerShell):
ssh-keygen

Result:
Generated ed25519 key pair:
- Private key: id_ed25519
- Public key: id_ed25519.pub

---

### Step 2 - Add Public Key to Raspberry Pi

Command (Windows PowerShell):
type $env:USERPROFILE\.ssh\id_ed25519.pub

Action:
Copied the public key and added it to the Raspberry Pi:

~/.ssh/authorized_keys

Commands (on Raspberry Pi):
``` bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

Result:
Public key successfully added for authentication.

---

### Step 3 - Configure Permissions

Commands:
``` bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Result:
SSH directory and authorized keys file secured with proper permissions.

---

### Step 4 - Verify Passwordless Login

Command:
ssh tomas683@192.168.1.38

Result:
Successfully logged in without being prompted for a password.

---

### Step 5 - Multi-Device Access

Additional device configured with its own SSH key.

Each device's public key was added to:
~/.ssh/authorized_keys

Commands used:
``` bash
nano ~/.ssh/authorized_keys
```

Permissions verified:
``` bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Result:
Multiple systems can securely access the Raspberry Pi using their respective SSH keys.

---

### Step 6 - SSH Configuration Hardening

Updated SSH configuration:

PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3

Command:
``` bash
sudo nano /etc/ssh/sshd_config
```

Applied changes:
sudo systemctl restart ssh

Result:
- SSH access restricted to key-based authentication only
- Root login disabled
- Authentication attempts limited to reduce brute-force risk

---

## Final Result

- SSH secured using key-based authentication
- Password-based authentication disabled
- Multiple trusted devices configured for access
- Root login disabled
- Authentication attempts limited to reduce brute-force attacks

System access is now restricted to authorized key-based clients only.