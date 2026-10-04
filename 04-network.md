## Network Configuration

### Goal
Configure Raspberry Pi to use a single, stable network interface for consistent and reliable operation.

---

### Step 1 - Review Network Interfaces

Command:
``` bash
nmcli connection show
```

Result:
Wired connection (eth0) present
No active Wi-Fi connection configured

Conclusion:
System is primarily using wired networking.

---

### Step 2 - Verify Interface Status

Command:
``` bash
ip -br a
```

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

### Step 4 - DHCP Reservation

Current State:
Static IP reservation for the Pi (192.168.1.38), ensuring the address never changes across reboots or DHCP lease renewals.

---

## Router Migration: Netgear Orbi → UniFi Dream Router 7

### Goal
Replace consumer mesh Wi-Fi (Netgear Orbi) with a UniFi Dream Router 7 (UDR7) to get real VLAN support, a proper zone-based firewall, and per-device visibility that the Orbi couldn't provide.

### Reason for the Change
The original setup (see Step 1-4 above) was built on the Orbi, which handled routing and Wi-Fi but offered no VLAN segmentation and only basic firewall rules. As the lab grew — IoT devices, a self-hosted media stack, a security camera NVR — keeping everything on one flat network became a real risk: a compromised smart plug would have had the same network access as the file server.

### What Changed
- **Router/gateway**: Orbi → UDR7 (UniFi Dream Router 7)
- **DHCP authority split**: Internal LAN DHCP is handled by Pi-hole (so client hostnames and DNS visibility stay tied to the DNS server, same reasoning as the original Orbi→Pi-hole migration in `06-dns.md`); IoT and Vivint VLANs are handled by the UDR7 itself, with Pi-hole set as their DNS server
- **Wireless**: Multiple SSIDs now map to different VLANs instead of one flat network (see below)
- **Firewall model**: Simple allow/block rules → UniFi's zone-based firewall (V2), described in `08-vlan-segmentation.md`

### Current Physical Topology
- UDR7 port 1: trunk port (carries every VLAN) to a downstairs unmanaged switch, which in turn connects the Pi and a second wireless access point
- UDR7 port 2: access port, native VLAN only — dedicated to the security system's NVR, physically isolated from everything else
- UDR7 port 3: access port, native VLAN only — dedicated to a single IoT hub device

### Lessons Learned
- **Unmanaged switches can still carry tagged VLAN traffic** — a switch doesn't need to understand 802.1Q to pass tagged frames through; it just needs the upstream device (the UDR7, or a device doing its own tagging) to handle the tagging. This matters later for a dedicated lab machine sharing that same unmanaged switch (see `08-vlan-segmentation.md`).
- **Don't trunk a port that's carrying a single-purpose device.** The NVR's port was deliberately kept as a plain access port rather than a trunk — converting it would have put the NVR's untagged traffic on the wrong network and silently broken video storage.
- **A single point of failure is still a single point of failure, even on "good" hardware.** The unmanaged switch downstairs carries both the Pi and the wireless AP — if it fails, the Pi (and everything depending on it, including DNS for the household) goes down even though the UDR7 itself is fine. Documented and accepted as a known tradeoff rather than something silently discovered during an outage.
