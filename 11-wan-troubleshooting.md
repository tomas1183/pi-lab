## Case Study: WAN Instability Investigation (ISP-Side, Status: Ongoing)

### Note on status
Unlike most of this repo, this one doesn't end with a fix — it ends with a documented, evidence-backed case for an ISP escalation. Included anyway because the investigation itself, and catching a monitoring blind spot along the way, is the real value here.

---

### The pattern
Starting mid-August, the WAN connection (Verizon Fios, on the UniFi Dream Router 7's WAN1 port) began dropping intermittently and self-resolving within seconds to tens of seconds. First noticed as isolated blips days apart; by the third occurrence, the UDR7 itself had escalated it to a high-severity recurring alarm on its own — not something flagged manually, the platform's own pattern-detection kicked in once it crossed a threshold.

### First response: add a monitor — then find the monitor was lying
Added an uptime monitor pinging a public address every 30 seconds, tuned so a single transient blip wouldn't alert but a real outage would. During an unrelated routine health check weeks later, found its **retry count had been misconfigured** — set to 20 retries instead of the intended 3. At the configured retry interval, that meant the monitor needed over six minutes of continuous, unbroken failure before it would ever alert.

Every outage observed up to that point had been under two minutes. **The monitor had been silently incapable of catching a single one of them.** Corrected the retry count — but this was the first real lesson of the investigation: a monitoring tool existing and looking "configured" is not the same as it actually being capable of catching the thing it was built for. The fix was simple; the miss had been sitting there undetected for weeks.

### Ruling out the home side
Before escalating to the ISP, checked what was actually controllable: the UDR7's own per-port error counters for that WAN port. Zero receive errors, zero transmit errors — no CRC or frame corruption, which is what a damaged cable or bad connector typically produces. That result pointed away from a simple physical fault on the home end and toward something upstream - either Verizon's local network, or the ONT (the fiber-to-Ethernet box Verizon supplies and controls) rather than anything on the home side.

### A frequency jump that changed the investigation's posture
One night, three separate outage/recovery cycles happened inside a single 35-minute window — a sharp jump from the prior roughly-daily cadence. At that point this stopped being a "monitor and see" situation and became something worth building a documented case around before calling the ISP, rather than just reporting "it's been dropping sometimes."

### Full-session tracking and a timeline-based root-cause argument
Ran a multi-hour overnight tracking pass capturing every flap with exact timestamps and durations. The result: 11 distinct outages, all clustered inside a roughly 107-minute window during peak evening usage hours, followed by several hours of completely clean connectivity afterward. The longest single outage was under 40 seconds; most were in the single digits.

That clustering pattern was itself the argument: a physically damaged cable or loose connector tends to fail constantly, or specifically when physically disturbed — not on a schedule tied to time of day. A tight cluster during peak-usage hours, followed by total silence overnight, points toward something load-dependent further upstream (shared local infrastructure, or the optical equipment itself), not a simple bad wire in the house. Combined with the clean error counters from earlier, this became the core of the case for an ISP-side investigation rather than a truck-roll to check house wiring.

### Written up and escalated
Compiled the full timeline, the error-counter evidence, and a specific, non-generic list of questions for the ISP (optical signal power levels on their equipment, their own line-side diagnostics — not "please reset my router") into a report for Verizon, referencing the UDR7's exact recurring alarm signature (its 'WAN failed multiple times' escalation). This reframes the ask from "something feels off" into "here's the data, please check these two specific things."

### It recurred again afterward
The same signature reappeared about two weeks later: four more flap cycles within roughly twenty minutes, surfaced that time through an actual symptom (streaming buffering) rather than a proactive check. Verified Wi-Fi and the relevant streaming service's own status page first — both clean — before confirming via the UDR7's own alarm log that this was the same recurring WAN issue, not something new. **Notably, the ping-based monitor missed all four of these too, even after being corrected** — reinforcing that for this specific failure mode, the UDR7's own alarm/event log is the authoritative source, and the lightweight ping monitor is only a secondary convenience check, not something to rely on for this particular problem.

---

## Lessons Learned

- **A monitor that exists is not the same as a monitor that works.** The retry-count misconfiguration sat silently wrong for weeks; only an unrelated routine check caught it. Any alerting threshold is worth occasionally testing against a known event, not just set-and-forget.
- **Clean error counters are themselves a finding, not a dead end.** Ruling out the home side with concrete data (zero rx/tx errors) is what turns "my internet drops sometimes" into an actionable, specific ISP escalation.
- **Timing patterns carry real diagnostic weight.** A tight cluster during peak hours, followed by total quiet, argues for a load-dependent upstream cause in a way that a vague "it happens sometimes" complaint never could.
- **When two monitoring sources disagree, trust the one closer to the actual event.** The platform's own alarm log caught every occurrence; the generic ping monitor caught none of them, even retuned. For this specific failure mode, that made the vendor's own telemetry the primary source of truth, not the general-purpose tool.
