## Case Study: Secrets Hygiene and Public Exposure Audit

### Context
Started as a single, narrow task — rotate a Pi-hole password that had been accidentally exposed while debugging something unrelated. It turned into a full audit of how every service on the host handled secrets, and a separate discovery about exactly what was reachable from the public internet.

---

## Part 1: Credential Rotation

### The trigger
A broad `grep` while investigating an unrelated issue printed Pi-hole's admin password in plaintext to the terminal. Straightforward enough: rotate it and move on.

### It kept breaking the same way
Pi-hole's password is only changeable via the `FTLCONF_webserver_api_password` environment variable — its own UI makes the field read-only once that variable is set. So the fix meant moving credentials for several stacks out of inline plaintext `environment:` values and into Portainer's stack-level "Environment variables" section instead, referenced via `${VAR}` substitution in the compose file.

Every single stack touched that night broke on the first attempt, always with the same symptom: the container came up but was non-functional, or crash-looped with an auth/key-missing error — while the value genuinely existed in Portainer's UI.

**Root cause, found by checking `docker inspect`'s actual resolved environment list rather than trusting the UI**: Portainer's environment-variable section only *substitutes into* a `${VAR}` placeholder that already exists in the compose text. It does not inject anything new. Each failure traced back to the compose file not actually referencing the variable correctly — a placeholder commented out, a typo'd variable name, a stray literal quote character typed into the value field, or (in one case) a variable written as `- KEY: value` inside a list-style `environment:` block, which is invalid YAML mixing a dict-style entry into a string list. Four separate stacks hit four different variants of the same underlying mistake.

**Fix applied uniformly**: go stack by stack, confirm the compose file's `environment:` block actually contains `${VAR_NAME}` for each secret, fix the specific defect, redeploy, and verify the resolved value with `docker inspect` rather than assuming the Portainer UI save was sufficient.

### A second, bigger exposure — during verification itself
While confirming the fixes above, a `docker inspect ... | grep -i <widget_prefix>` command — intended to check one variable — matched a shared naming prefix and printed **six separate credentials** to the terminal at once: a long-lived Home Assistant token, an NPM admin password, a UniFi API key, the Pi-hole password (again), and a Portainer API key.

**Response**: rotated all six, plus one more exposed separately via a screenshot during the same session. Each rotated value was pushed back into both its source service *and* Homepage's dashboard widget config, then verified live — reloading the dashboard and confirming every widget (Portainer, NPM, UniFi, Pi-hole, Home Assistant, Tailscale) still pulled live data with the new credentials, rather than assuming the rotation alone was sufficient.

### What was deliberately *not* rotated — and why that mattered
Two other values came up during the same pass and were left alone on purpose:
- A media-stack service's internal secret key, which its own documentation states explicitly cannot be changed after first use without orphaning every existing stored user config tied to it (shared with a family member's setup) — and which had, on inspection, never actually been exposed in plaintext to begin with, only shown once as a partially masked value.
- A backup tool's settings-encryption key, which decrypts that tool's own existing job configuration — same risk profile as the above.

Reflexively rotating *everything* that was merely adjacent to an exposure would have been the less careful choice here, not the more secure one — the right call required checking the specific service's own semantics first, not just reacting to the word "exposed."

---

## Part 2: Public Exposure Scoping

### A separate, unrelated discovery
While auditing dashboard tile links for a different reason, it surfaced that the reverse proxy (nginx-proxy-manager) had public DuckDNS proxy-host entries for far more services than intended. Only two services were ever meant to be reachable from the internet; in practice, five more were too — including the Docker management UI and the backup tool's own web UI.

### Why this was a real gap, not just messy config
The router's port-forward for 80/443 isn't scoped to a specific hostname — it just hands matching traffic to the reverse proxy, which then matches purely on the HTTP `Host` header. That means "this address was never shared with anyone" is obscurity, not protection: anyone who guessed or found the hostname pattern had a working path straight to full container-management control, not just the two services that were supposed to be public.

### Fix
Confirmed which two services genuinely needed to stay public (a remote family member relies on one of them with her own separate configuration, so pure private/Tailscale-only access wasn't an option for that one). Disabled — not deleted — every other public proxy-host entry, dropping those services to LAN/Tailscale-only while keeping the configuration intact for a fast, low-risk re-enable if ever needed again.

---

## Lessons Learned

- **A setting that "should" take effect isn't confirmed until read back from the running container.** Four stacks failed the same way because the UI's saved value and the compose file's actual reference silently disagreed — `docker inspect` catches that gap; assuming the save worked doesn't.
- **Scope verification commands as narrowly as the thing you're actually checking.** A broad grep by shared prefix, meant to check one value, printed six unrelated credentials at once. Searching for an exact variable name instead of a prefix is the real control here, not "being more careful" in the abstract.
- **Rotation isn't automatically the safe default.** A credential tied to data that can't survive rotation needs its own judgment call, not a blanket "exposed, so rotate it" response.
- **A port-forward that isn't hostname-scoped exposes everything behind it, not just the service you meant to expose.** "Nobody knows this URL" is not a security boundary when the matching logic (Host header) is sitting right behind an open port.
