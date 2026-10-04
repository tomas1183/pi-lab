## Reverse Proxy: Friendly Internal Hostnames

### Goal
Replace IP:port addressing (`192.168.1.38:9000`) with friendly hostnames (`portainer.framboise`) for every internally-used service, without making anything newly reachable from the public internet.

---

### Architecture

- **Pi-hole local DNS records** map each `*.framboise` hostname to framboise's own LAN IP (`192.168.1.38`) — not a Tailscale IP. This was a deliberate choice: a Tailscale IP is only reachable by devices actually running the Tailscale client, which would have broken access for any plain LAN device (phones, TVs, other household members) not on the tailnet. Remote/away access was already solved separately via Tailscale MagicDNS (`http://framboise:<port>`) — this project was about killing port numbers for local use, not building a new remote-access path.
- **NPM Proxy Hosts** — one per hostname, each forwarding to the service's real LAN IP:port.
- **NPM Access List ("LAN + Tailscale Only")** applied to every proxy host created here: allows `192.168.1.0/24` (LAN) and `100.64.0.0/10` (Tailscale CGNAT range), denies everything else. This is the control that actually keeps these internal — the public port-forward routes by Host header regardless of hostname, so the access list (not just "nobody publicized the name") is what makes these non-public.
- **Plain HTTP, no TLS cert.** `.framboise` isn't a real public domain, so the usual HTTP-01 cert method doesn't apply, and NPM's DNS-01 DuckDNS integration has open reliability issues upstream. Skipped entirely for this pass — self-signed certs (one-time CA trust per device) remain the documented fallback if browser warnings ever become annoying enough to revisit.

---

### Discovery: half of this already existed, unused

Before building anything, found that Pi-hole already had local DNS records for 5 of these hostnames, dating back months, sitting completely unused — the NPM side had never been built, so visiting any of them just landed on NPM's default page. An earlier, undocumented attempt at this same project had apparently been half-started and abandoned. Finished it rather than redoing it from scratch.

---

### Hostname → Port Mapping

All forward to `192.168.1.38`:

| Hostname | Service | Port |
|---|---|---|
| pihole.framboise | Pi-hole | 8080 |
| portainer.framboise | Portainer | 9000 |
| stats.framboise | Glances | 61208 |
| dash.framboise | Homepage | 85 |
| ha.framboise | Home Assistant | 8123 |
| npm.framboise | Nginx Proxy Manager (its own UI) | 81 |
| kuma.framboise | Uptime Kuma | 3001 |
| duplicati.framboise | Duplicati | 8200 |
| prowlarr.framboise | Prowlarr | 9696 |

### Deliberately not given a friendly name
- **UniFi/UDR7** — a separate device with its own IP and certificate; folding it into framboise's naming scheme would be a conceptual mismatch.
- **Tailscale** — the dashboard tile just links to Tailscale's own cloud admin console; nothing self-hosted to proxy.
- **Stremio/AIOStreams** — already intentionally public via a separate DuckDNS domain; adding an internal-only name too would be low-value, not harmful if ever wanted later.

---

### Gotcha: Homepage rejected its own new hostname

`dash.framboise` returned HTTP 400 "Host validation failed" the moment its NPM proxy host went live. Homepage (Next.js-based) checks incoming requests against a `HOMEPAGE_ALLOWED_HOSTS` environment-variable allow-list as a CSRF protection, and that list only had the old port-based host strings. Fixed by adding `dash.framboise` to the list — but env var changes require a full container recreate to take effect, not a live config reload, which cost an extra round-trip before the fix actually applied.

**How to apply if this happens again**: any other Next.js-based service added to this reverse proxy should be checked for the same kind of host-header allow-list before assuming a 400 means a proxy misconfiguration.

### Gotcha: a `sed` substring collision while fixing dashboard tile links

Once `dash.framboise` resolved on its own, its dashboard *tiles* still linked to the old port-based URLs — fixing that meant a batch `sed` pass across `services.yaml` replacing each `framboise:<port>` with its new friendly name. One rule targeting `framboise:81` (NPM) silently partial-matched inside `framboise:8123` (Home Assistant), since `sed` does plain substring matching with no word boundary — producing a mangled `npm.framboise23` instead of leaving Home Assistant's link alone. Caught only by eyeballing the full `grep href:` output after the edit, not automatically.

**How to apply**: when chaining several numeric-port `sed` replacements where one port is a prefix of another (`81` vs. `8123`, `80` vs. `8080`), either order the replacements longest-pattern-first or anchor the match (trailing `/` or line-end) instead of trusting the replacements to be independent.

### Gotcha: `sed -i` failed with "Permission denied" despite a writable file

The YAML file itself was owned by the regular user, but the *containing directory* was root-owned — and `sed -i` creates its replacement via a temp file in that same directory before renaming it into place, which fails even though the target file is directly writable. Worked around by redirecting to `/tmp` first, then overwriting the real file via shell redirection rather than letting any tool create-and-rename inside that directory.

---

### NPM API automation pattern (reusable)

Rather than clicking through NPM's UI for 9 near-identical proxy hosts, scripted it against NPM's own API: credentials pulled via grepping the two exact environment-variable names needed (never a broad `docker inspect`/env dump — a past incident had already shown how easily that over-exposes unrelated secrets), then `POST /api/tokens` with `{identity, secret}` to get a short-lived JWT, used as a bearer token for the subsequent `/api/nginx/proxy-hosts` calls.

---

## Lessons Learned

- **Half-finished infrastructure is easy to miss without checking state first.** The unused Pi-hole DNS records sitting there for months were a reminder to check what already exists before assuming a project is starting from zero.
- **"Internal-only" is enforced by the access list, not by obscurity.** The same public port-forward sits in front of every proxy host regardless of hostname; the IP-range access list is the actual control.
- **An environment-variable change and a live config change are not interchangeable** — some fixes need a full recreate, others hot-reload, and assuming the faster one applies to both costs a confusing extra debugging pass.
- **Plain-text substring tools like `sed` need explicit boundaries once two values share a prefix.** A quiet partial match is worse than an outright failure, since nothing flags it — only a manual diff caught it here.
