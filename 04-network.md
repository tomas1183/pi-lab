## Network Configuration

### Goal
Configure Raspberry Pi to use a single, stable network interface for consistent and reliable operation.

---

### Step 1 - Review Network Interfaces

Command:
nmcli connection show

Result:
Wired connection (eth0) present  
No active Wi-Fi connection configured  

Conclusion:
System is primarily using wired networking.

---

### Step 2 - Verify Interface Status

Command:
ip -br a

Note:
Used `ip -br a` for a simplified view of network interfaces.

Result:
eth0    UP    192.168.1.38/24  
wlan0   DOWN  

Conclusion:
- Wired interface (eth0) is active  
- Wireless interface (wlan0) is inactive  

---

### Step 3 - Ensure Single Interface Usage

Observation:
Wi-Fi interface exists but is not active and has no IP address assigned.

Decision:
Use only eth0 (wired) for network connectivity.

Reason:
- More stable connection  
- Avoids multiple IP addresses  
- Prevents DNS conflicts  
- Best practice for server environments  

---

### Step 4 - DHCP Configuration

Current State:
IP address assigned dynamically via DHCP:
192.168.1.38

---

### Step 5 - DHCP Reservation Verification

Existing DHCP reservation was previously configured in the Orbi router.

Verification steps:
- Confirmed IP address assignment (192.168.1.38)
- Rebooted system to ensure IP persistence

Result:
IP address remained consistent after reboot, confirming DHCP reservation is active.

---

## Final Result

- System uses only wired interface (eth0)
- Wi-Fi interface present but not configured or active
- Single IP address assigned
- DHCP reservation ensures consistent IP address
- Network configuration is stable and predictable

System is ready for network-dependent services such as Pi-hole.