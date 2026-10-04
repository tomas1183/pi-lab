## Case Study: Wi-Fi Mesh Backhaul and RF Interference

### Context
Started while investigating streaming buffering on a bedroom smart TV — initially suspected it might be the same recurring WAN issue (see `11-wan-troubleshooting.md`), but this turned out to be an entirely separate, local problem with a clean resolution.

---

### Finding the real cause
The UDR7 (main router) and the U7 Pro (a mesh access point extending Wi-Fi to another part of the house) were both operating their 5GHz radio on the exact same channel. The mesh AP's own backhaul to the router was itself wireless — meaning every frame for every client connected to that AP crossed the same congested airspace twice: once for the backhaul, once for the client traffic.

Measured the impact directly rather than guessing: the mesh AP's 5GHz transmit-retry rate during an active streaming session was running around **20%**, against a known-clean baseline of about 7% measured when the AP was first positioned. The TV's own link rate degraded mid-session while connected. Notably, the router's built-in "client satisfaction" score stayed at 99–100 the entire time — that metric turned out to be useless for this failure mode; transmit-retry rate was the number that actually told the story.

### Fix: hardwire the backhaul
Ran an existing Ethernet cable to convert the mesh AP from wireless to wired backhaul, freeing its 5GHz radio to serve clients only. This needed one non-obvious decision: **which switch port to use.** The router's ports weren't interchangeable — one was already a full trunk carrying every VLAN with the main network untagged (what an access point actually needs: untagged management plus tagged SSID traffic), while the others were locked to single-purpose VLANs for other devices and would have put the AP on the wrong network, or broken a different single-purpose device, if used instead.

**Result, verified immediately**: the AP's uplink flipped from wireless to a solid wired connection, its 5GHz radio moved off the shared channel automatically (no longer forced to match the router's channel to maintain the wireless mesh link), and channel utilization on that radio dropped from roughly 30% to 6%.

### An unexpected side effect: a DFS trap
The AP's automatic channel selection, now free to choose anything, picked a channel that falls inside a DFS (Dynamic Frequency Selection) band. Some older Wi-Fi hardware — specifically a couple of older Wi-Fi 5 smart TVs in this case — doesn't support DFS channels at all. The practical effect: those specific TVs couldn't see the near AP's 5GHz network anymore and silently fell back to the far-away router instead, landing on a much weaker signal and lower link rate than before the "fix."

**Diagnostic tell**: newer devices (phones, laptops) had no problem on the new channel; only the older TVs were affected. That split — not "everything got worse," but "specific older devices got worse" — pointed straight at a DFS compatibility gap rather than a general RF problem.

**Fix**: manually pinned the AP's 5GHz radio to a known non-DFS channel. The affected TV immediately reassociated with a dramatically better signal and link rate than it had even before this whole investigation started.

### Chasing a slower fix: the channel was still noisy under load
Even after the DFS fix, a later check under real streaming load showed the pinned channel's retry rate climbing into the 10–16% range during active use — not as bad as the original shared-channel problem, but still higher than the original clean baseline, and the pattern (retries scaling with load, not present at idle) pointed at interference rather than simple congestion.

Rather than guess, ran an actual RF spectrum scan from the access point itself. The scan showed the channel block in use spanned four adjacent 20MHz slices, and one of those specific slices had a strong, clearly external interference source — almost certainly a neighboring network — raising the noise floor for the entire wider channel built on top of it.

**Fix**: narrowed the channel width just enough to exclude the specific noisy slice while keeping the two clean ones, still avoiding the DFS range entirely. Verified afterward under the same load conditions: signal and reported satisfaction both back to clean, best-case numbers.

**Worth noting on cost**: narrowing a channel's width sounds like giving something up, but mapping out which devices actually use that radio first (one older TV and one laptop, never more than a single simultaneous 4K stream in practice) showed there was no real capacity being sacrificed — the narrower channel still had far more headroom than what those specific devices could ever use at once.

---

## Lessons Learned

- **A reassuring aggregate "satisfaction" score can hide a real problem.** It read 99–100 throughout the entire original contention issue. Transmit-retry rate, measured directly, is what actually exposed it — and what should be checked first for this class of complaint going forward.
- **Freeing up a resource (a radio, a channel) can immediately create a new, different problem.** The DFS trap only existed *because* the hardwiring fix worked and let the radio pick its own channel — a good fix still needs its own follow-up verification, not just "it moved to a new channel, done."
- **A split symptom (some devices affected, others not) is itself a diagnostic clue**, not just noise — it was the detail that pointed directly at a hardware-support gap (DFS) rather than a general signal problem.
- **Don't guess at RF interference — scan for it.** The specific noisy sub-channel wouldn't have been found by reasoning alone; an actual spectrum scan turned "it's still a bit flaky" into an exact, fixable cause.
- **Stale Wi-Fi associations don't self-correct.** A client holding a working-but-bad connection doesn't rescan on its own after a channel change — it has to be explicitly disconnected to force it to re-evaluate and pick the better option.
