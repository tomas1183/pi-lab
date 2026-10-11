## Case Study: Standing Up a Dedicated Windows Server Identity Lab (Status: In Progress)

### Context
Part of a Microsoft identity-focused certification path: AZ-900 (Azure Fundamentals) completed, AZ-104 (Azure Administrator) in progress, SC-300 (Identity and Access Administrator) planned next. SC-300's hybrid-identity objectives — on-premises Active Directory synced to a cloud identity tenant, Conditional Access interacting with that sync — are the one part of this path that's genuinely replicable in a homelab. Azure administration itself (AZ-104) is mostly cloud-native and needs a real subscription, not self-hosted infrastructure; but a real on-prem Active Directory Domain Services forest, later paired with a cloud sync tool, is exactly how hybrid identity works in actual production environments.

### Why a separate machine at all
The Raspberry Pi running the rest of this homelab is ARM — it cannot run Windows Server under any circumstances. A dedicated x86 machine was the only way to get real hands-on AD DS experience at home. A retired 2010-era laptop (originally carrying a dead battery and a non-genuine Windows 7 install, both dealt with before this box went anywhere near the network) became that machine, repurposed specifically for this.

### Goal
Stand up a fully isolated Windows Server domain controller — its own dedicated VLAN and zone-based firewall policy (see `08-vlan-segmentation.md`) — to get real hands-on practice with AD DS now, and Entra Connect hybrid-identity sync next.

### Platform
- Repurposed 2010-era laptop, RAM upgraded, previous drive wiped and replaced
- Windows Server 2025 Standard Evaluation, installed as **Server Core** (no GUI) specifically to minimize footprint on old hardware — managed entirely remotely afterward via WinRM and RDP, never the laptop's own screen

---

## Case Study: The NIC That Couldn't Tag VLANs

The laptop's physical isolation depends on 802.1Q VLAN tagging happening at the server itself (the switch in the path is unmanaged and can't assign VLANs by port — see `08-vlan-segmentation.md`). The Ethernet adapter's own driver turned out to have no VLAN ID field at all — confirmed directly by inspecting its advanced properties, not assumed — only a generic "priority tagging pass-through" toggle, which isn't the same thing.

The obvious next step, NIC Teaming (which lets you add a VLAN-tagged virtual adapter on top of any physical NIC regardless of driver support), turned out to be a dead end too: it's specifically deprecated in this version of Windows Server, with real-world reports of it failing outright on fresh installs. Not worth using even though it still technically exists.

**The actual working fix**: install the Hyper-V role — not to run any virtual machines, but purely to get access to an external virtual switch, which supports VLAN tagging in its own networking stack independent of the physical NIC driver. Tagging the *management OS's own connection* to that virtual switch gave a clean, working VLAN-tagged interface. Worth being upfront about the tradeoff: this gives the box a full virtualization role it will never otherwise use, in exchange for a working, properly supported tagging mechanism — accepted deliberately, not an oversight.

Verified afterward that the upstream network trunk was already configured to automatically carry any newly-defined VLAN, so no additional router-side changes were needed once the tagging was correctly applied on the server side.

---

## Case Study: A Naming Collision During Forest Promotion

The first attempt to promote the server to a new Active Directory forest failed prerequisite validation outright, citing a NetBIOS name conflict. The cause: the domain's intended NetBIOS name had been set to the exact same string as the computer's own existing NetBIOS name.

**The lesson**: a domain's NetBIOS name and an individual machine's NetBIOS name share the same flat namespace and cannot be identical — worth deliberately picking a distinct NetBIOS name any time a server's hostname happens to match the first label of the domain it's about to create. Fixed by choosing a different NetBIOS name for the domain; the actual DNS domain name was unaffected.

---

## Case Study: Reachable by Firewall Rule, Still Blocked

With the VLAN-isolated network's firewall confirmed to correctly allow management traffic in (checked directly against the live policy set, not assumed), the server's remote-management service was still completely unreachable.

The cause turned out to be a second, independent layer: the new network connection had defaulted to the **Public** network security classification. Server Core has no graphical prompt to let you choose a network location on first connect the way desktop Windows does, so it silently defaults to the more restrictive setting — and the OS's own built-in firewall rule group for remote management only applies automatically to Private/Domain-classified connections, not Public ones.

Confirmed the actual classification directly, corrected it, and verified the fix by testing real port reachability from another machine on the network before and after — rather than assuming success from the command completing without error.

One practical wrinkle: the service that needed fixing (WinRM) obviously couldn't be used to fix itself. A separate remote-access method (RDP, on its own distinct firewall rule, unaffected by the same block) was the way in to actually apply the correction.

**The broader lesson**: two systems can each independently report "working as configured" — the network firewall was correctly set up, and the server reported no errors — while the actual end-to-end path still fails, because the real blocker was a third, independent layer neither side was individually responsible for.

---

## Case Study: A Default-Deny Zone With No DNS Path At All

Once the server was given a static IP pointing at the household's internal DNS resolver, every single DNS lookup failed outright — including, later, the online activation step for the server's own license.

Traced directly against the live zone-based firewall policy set rather than guessing: the isolated zone had narrow, correctly-scoped inbound rules for remote management, but genuinely nothing allowing *any* outbound traffic toward the rest of the internal network — DNS included. This wasn't a regression; it was a gap that had simply never been exercised before, since nothing in that zone had needed to reach anything else internally until this point.

**Fixed** with one narrow, specific rule — that zone, to the DNS server's exact IP, port 53 only — mirroring the identical pattern already used elsewhere in this network for a different isolated segment's DNS exception. A default-deny security posture does exactly what it's designed to do, including blocking dependencies nobody thought to explicitly allow yet. Worth treating every new outbound requirement from an isolated zone as something that needs its own deliberate rule, not something assumed to "just work."

---

## Case Study: An Assumption About License Limits, Caught by Verification

Planned a small local automation — a scheduled task that checks the evaluation license's remaining time and, when it's running low, automatically renews it ("rearms" it) and sends a notification — based on a commonly-cited figure that this specific evaluation channel supports renewing up to six times.

Wanted to confirm the automation's notification actually worked before trusting it to run unattended for months, so ran a one-off manual test. The test script didn't cleanly separate "confirm the notification fires" from "actually execute the renewal" — both lived in the same code path — so running the test consumed a real renewal, not a simulated one.

Checking the license status directly afterward (rather than continuing to trust the earlier figure) showed the real number: **zero renewals remaining** for this installation, not six. The commonly-cited figure was wrong for this specific evaluation channel.

**The lesson, reinforced twice in one evening** (it also applied to the firewall-policy checks above): verify a stated limit or count directly against the live system before building a plan around it, and never assume a "quick test" of something with a real side effect is actually free without first isolating that side effect.

**Practical outcome, not a crisis**: the current license period is fully valid and unaffected. The next expiration will require a clean reinstall rather than another renewal — a fully acceptable, already-planned-for outcome for a lab machine, with a documented, scripted reinstall procedure staged and ready for exactly that scenario.

---

## Verification Before Moving On

Rather than trusting the promotion wizard's own "success" message, ran the standard domain-controller health diagnostic afterward. Every substantive test passed — connectivity, service health, replication readiness, all directory partitions, site and locator checks. Two tests reported as "failed" were both well-understood false-positive patterns specific to a brand-new, single-domain-controller forest (one needs a second, replication-partner controller to have a real finding to report; the other flags *any* benign log entry regardless of actual relevance) rather than genuine problems.

---

## What's Next

**Entra Connect** — pairing this on-premises forest with a cloud identity tenant to get real hands-on practice with the actual hybrid-identity sync mechanisms SC-300 tests. Not yet built; this document will be extended once that work is done.

---

## Lessons Learned

- **A hardware limitation doesn't always have the expected fix.** The straightforward-seeming workaround (NIC Teaming) turned out to be the wrong path on this specific OS version; the real solution came from an entirely different subsystem (Hyper-V's own virtual switch layer) than where the original problem lived.
- **Two independently "working" systems don't guarantee a working end-to-end path.** The network firewall being correctly configured and the server reporting no errors were both true at the same time as the actual connection still failing — because of a third, independent layer neither side owned.
- **Default-deny security does exactly what it's built to do, including to things you haven't thought of yet.** Every new outbound dependency from an isolated network needs its own deliberate rule; nothing should be assumed to "just work" by default.
- **Never trust a remembered number for something with a real, limited, non-renewable cost without checking it live first** — and don't assume a test of an action with real side effects is free unless that side effect is explicitly isolated or confirmed low-cost first.
