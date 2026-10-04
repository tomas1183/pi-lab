## Monitoring: Uptime Kuma and Glances

### Goal
Get real visibility into whether every service is actually working — not just whether its container shows "running" — after being burned by exactly that distinction once already.

### Why this exists
A security-related container (a CrowdSec firewall bouncer) was discovered to have been silently crash-looping for roughly 52 hours before anyone noticed. `docker ps` would have shown it restarting, but nothing was actively watching for that pattern, so it went unnoticed until a routine check happened to catch it. That incident is the direct reason this monitoring stack exists — the goal isn't just "is the container up," it's "is the thing the container is supposed to be doing actually happening."

---

## Platform

- **Uptime Kuma** — status monitoring and alerting, 13 monitors as of now, all feeding a shared notification channel.
- **Glances** — live host/container resource monitoring (CPU, memory, per-container stats), no auth required for its own dashboard widget.

---

## Monitor Type Strategy

Kuma supports several monitor types, and picking the wrong one for a given service means passing checks that don't actually verify anything useful. The rule used here: **default to an HTTP monitor for anything with a real web UI or API** (most services) — but switch to a different type whenever HTTP wouldn't actually test the thing that matters:

| Type | Used for | Why HTTP wasn't right |
|---|---|---|
| HTTP | Most services with a web UI | (default case) |
| Ping | WAN/internet reachability | There's no HTTP endpoint that represents "is the internet up" |
| TCP-Port | CrowdSec firewall bouncer | No web UI at all — just needs to confirm the port it listens on is actually open |
| DNS | Full Pi-hole → Unbound resolution chain | Pi-hole's web UI being up doesn't prove DNS resolution actually works end-to-end; a DNS-type check that resolves a real domain through the resolver does |
| HTTP(s) - Keyword | Prowlarr, FlareSolverr | A plain 200 OK isn't enough — both have failure modes where the page loads fine but the actual function is broken, so the check also requires a specific keyword in the response body |

### Gotcha: a redirect can make a broken check pass
Prowlarr's root URL (`/`) returns a 302 redirect, and Kuma follows up to 10 redirects by default — so a plain HTTP monitor pointed at `/` would report healthy even if the real underlying page failed, since it's really just checking that the redirect target also responds. Fixed by pointing the monitor at Prowlarr's dedicated `/ping` endpoint instead, which needs no auth and returns a direct, unambiguous `{"status":"OK"}`.

### Gotcha: "Keyword" isn't a field, it's a different monitor type
In this version of Kuma, there's no way to just add a keyword check to a regular HTTP monitor — Keyword is its own separate entry in the Monitor Type dropdown, and the keyword input box only appears after switching to it. Easy to miss since every other setting (URL, interval, retries, status-page membership) carries over unchanged when switching types, so it looks like the same monitor with one new field rather than a type change.

### Why FlareSolverr has its own monitor despite having no user-facing UI worth linking to
FlareSolverr is a dependency two other services route through silently — specifically, a couple of media indexers rely on it to get past anti-bot challenges on some source sites. If FlareSolverr dies, those indexers and the dashboard that surfaces them all stay green, because nothing about their own health checks would notice — they'd just quietly stop returning results from certain sources. The FlareSolverr monitor exists specifically to catch that class of failure: a dependency that's invisible everywhere except in the actual data it should have produced.

---

## Case Study: The Stale Docker-Socket Connection

### The problem
Three containers — Portainer, Watchtower, and Glances — all talk to the Docker API by bind-mounting the host's Docker socket directly, and all three hold that connection open as a long-running process rather than reconnecting per request. Any disruptive event on the host's Docker side (a heavy `docker build`/`docker rm` sequence, or the Docker daemon itself restarting) can leave any or all of them holding a dead connection to an otherwise completely healthy daemon. Running `docker ps` from a fresh SSH session worked fine throughout every one of these incidents — only the long-lived containers were affected, which is exactly what made it easy to miss.

### The symptoms varied — and one of them was silent
- **Portainer** showed "Down" in its own UI along with repeated connection-refused notifications — loud and obvious.
- **Watchtower** failed its next scheduled update check with a clear "cannot connect to the Docker daemon" error — also loud, just on a delay until its next cycle.
- **Glances failed completely silently.** Its API endpoint kept responding normally, just reporting zero containers, with nothing logged anywhere to indicate a failure. Nothing about this state looks like an error — it looks like "there happen to be no containers right now," which is actively misleading rather than just quiet.

### First fix: manual, but with a real root cause identified
A simple `docker restart` on the affected container(s) always fixed it immediately — the socket itself was fine the whole time, only the long-held connection was stale. The real root cause, once traced: these are the only three containers on the host with that specific bind-mount-and-hold-open pattern, which made the rule for predicting a future occurrence clean — this exact trio, and only this trio, was ever at risk.

### Second occurrence, and the permanent fix
The same failure class recurred after a routine automatic nightly reboot — not just the originally-identified `docker build`/`docker rm` trigger, but any event that restarts the Docker daemon itself, which a full reboot always does. This time, a dashboard widget relying on Portainer's API threw a request error, surfacing the same underlying issue through yet another symptom.

Fixed manually again, then closed the gap permanently: a small script that waits for each of the three containers to report as actually running (polling rather than assuming an instant restart), then restarts all three in sequence, wired into a systemd service configured to run once after the Docker service itself comes up on every boot. Verified working both on a manual trigger and automatically across a real subsequent reboot.

---

## Lessons Learned

- **A container reporting "running" and a container doing its job are two different claims.** Glances proved this concretely — fully "up," fully wrong, and completely silent about it.
- **The specific technical reason a failure class exists (a long-held connection vs. a healthy daemon) is what makes it possible to predict exactly which other containers share the same risk**, rather than treating each incident as a one-off surprise.
- **A monitor needs to test the specific thing that can quietly break, not just "does this respond."** A redirect-following HTTP check, a plain container-up check, and a keyword-less health check all would have missed a real failure in this stack at some point — the fix each time was matching the check type to the actual failure mode, not just adding more monitors.
- **Manually fixing a recurring issue twice is the signal to automate it, not a reason to document the manual steps better.** The systemd unit removes the need for anyone to notice and intervene at all, which is strictly better than a well-documented manual fix for something that will keep recurring on a predictable trigger (every reboot).
