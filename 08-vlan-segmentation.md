## Network Segmentation - VLANs and Zone-Based Firewall

### Goal
Segment the network by trust level so that a compromised or misbehaving device on one segment can't reach devices on another — specifically isolating IoT/smart-home devices, a third-party security system, and (most recently) an identity-lab test environment from the main household network.

---

## Architecture

### Networks

| Network | VLAN | Subnet | DHCP Authority | Purpose |
|---|---|---|---|---|
| Internal LAN | native (untagged) | 192.168.1.0/24 | Pi-hole | Trusted workstations, phones, servers |
| IoT | 2 | 192.168.10.0/26 | UDR7 | Smart-home devices (plugs, sensors, hubs) |
| Vivint | 3 | 192.168.20.0/29 | UDR7 | Third-party security system (NVR, panel) |
| HomeLab | 4 | 192.168.30.0/28 | UDR7 | Isolated lab environment for a dedicated test server |

### Why Pi-hole owns Internal LAN's DHCP but the UDR7 owns the others
Internal LAN DHCP was kept on Pi-hole (see `06-dns.md`) so trusted-device hostnames and DNS stay tied together. IoT and Vivint were deliberately left on the router's own DHCP server instead — these are lower-trust networks, and there's no benefit to Pi-hole knowing their individual hostnames; they're pointed at Pi-hole only for DNS resolution, not DHCP authority.

### Zone-Based Firewall
UniFi's firewall model groups networks into **zones**, then applies allow/block policy between zone *pairs* rather than per-device rules. Each VLAN above lives in its own zone. The default policy between any two zones is **block all**, with narrow, explicit exceptions added only where a real dependency exists. Examples of exceptions actually in place:
- IoT → Internal, port 53 only (so IoT devices can reach the Pi-hole DNS server, but nothing else on the trusted network)
- Internal → IoT, specific ports only (device casting/control protocols, like a phone reaching a smart display)
- HomeLab → Internal, blocked by default, with a narrow exception added for remote management (see below)

This "default-deny, explicit-allow" model means a new VLAN is isolated from everything by default the moment it's created — nothing has to be remembered or manually locked down later.

---

## Case Study: Standing Up an Isolated Lab Network

### Goal
Add a new, fully isolated VLAN for a dedicated test server (a repurposed laptop) without it having any path to the rest of the household network, while still allowing the server to reach the internet and be managed remotely from a trusted workstation.

### Step 1 — Create the Network
Configured a new network on the controller:
- Name: HomeLab
- VLAN ID: 4
- Subnet: 192.168.30.0/28 (right-sized for a single server plus a handful of test clients — not a full /24, which would have been needlessly large for this purpose)
- Network isolation: enabled

### Step 2 — Assign a Firewall Zone
UniFi ships with a fixed set of firewall zones (Internal, External, Gateway, VPN, Hotspot, DMZ, plus any custom ones defined). The first instinct was to use the built-in **DMZ** zone, since it was unused — but on reflection, that was the wrong call. A DMZ zone's entire purpose in networking is to be the segment you *expose* to the internet; reusing that label for an internal lab network is a mismatch that invites a future mistake (e.g., accidentally forwarding a port into "DMZ" without remembering what's actually living there). **Created a dedicated "HomeLab" zone instead** and reassigned the network to it.

### Step 3 — Verify the Isolation, Don't Assume It
After creating the network and assigning the zone, pulled the live firewall policy set directly rather than trusting that the configuration "should" be isolated:
- Confirmed HomeLab zone has an explicit block rule against every other zone (Internal, IoT, Vivint, Hotspot) in both directions, including against itself (devices within the lab network can't reach each other either)
- Confirmed outbound internet access (HomeLab → External) is allowed, and the gateway relationship (DHCP/DNS) is intact
- Confirmed no port-forward rule anywhere in the configuration targets this subnet, so nothing from the internet can reach in

### Step 4 — Add a Narrow Management Exception
The whole point of full isolation conflicted with one real requirement: the lab server needs to be managed remotely (via remote shell access) from a trusted desktop on the Internal LAN. Rather than opening the zone back up generally, added two specific, minimal exceptions:
- Internal → HomeLab, TCP port 5985/5986 (remote management protocol)
- Internal → HomeLab, TCP port 3389 (remote desktop, as a fallback option)

Everything else between those two zones remains blocked by default.

### Step 5 — Connect a Device Through an Unmanaged Switch
The lab server needed to physically connect through an existing unmanaged switch that also carries the Pi and a wireless access point — a switch with no VLAN awareness of its own. Since an unmanaged switch can't assign VLANs by port, the server itself was configured to tag its own traffic with VLAN 4 at the network-driver level. As long as the upstream trunk port on the UDR7 is already configured to carry that VLAN (it was, from Step 1), this works cleanly with zero changes needed on the switch itself — it just passes the tagged frames through without needing to understand them.

---

## Lessons Learned

- **Match zone names to intent, not convenience.** Reusing a zone just because it happens to be unused (like DMZ) can create a naming trap for future changes, even if nothing is actually misconfigured today.
- **Verify isolation by reading back the actual policy set, not by trusting a toggle's name.** A setting called "network isolation" doing what you assume it does is a reasonable guess, not a confirmed fact — checking the real, resulting firewall rules caught this reasoning gap before it mattered.
- **Full isolation and "I still need to manage this thing" are in direct tension — solve it with the narrowest possible exception**, not by reopening the whole zone. Two specific allowed ports is a much smaller attack surface than "Internal can reach all of HomeLab."
- **An unmanaged switch is not a barrier to VLAN segmentation** — the tagging can happen at the endpoint device itself instead of at the switch, as long as the upstream router port is already trunked correctly.
