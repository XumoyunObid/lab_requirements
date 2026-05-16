# Hard Machine Template — Constraints & Expectations

## Difficulty Profile

A **Hard** CTF machine targets strong intermediate and advanced players. It requires realistic multi-step chaining, deeper enumeration, and disciplined decision-making without relying on contrived shortcuts.

---

## Constraints

### Attack Chain
- **Exactly 4-5 steps:** Enumeration → Initial Access → Lateral Movement(s) → Privilege Escalation
- At least **one lateral movement** is required
- At least one step must require combining **two or more discoveries**
- Repeating the same vulnerability class across steps is discouraged unless strongly justified by architecture

### Initial Access
- Must use a modern, realistic vulnerability class or recent CVE in the exposed service
- Exploitation should require understanding request flow, auth logic, or deployment context
- Pure "run PoC and win" paths are not enough for Hard

### Lateral Movement (Required)
- Must cross a real trust boundary (user/service/network segment)
- Must come from realistic artifacts: service tokens, internal APIs, process metadata, deployment configs, or cracked hashes
- Must not depend on planted plaintext credentials in `.env`, backup files, notes, or hint files

### Privilege Escalation
- Must require system understanding, not one-command privilege escalation
- Should involve automation/deployment abuse (systemd, CI/CD, IaC, containers, package hooks) or a multi-condition local weakness
- The privesc path should be discoverable through methodical enumeration from the gained foothold

### Rabbit Holes
- **Exactly 2 rabbit holes**, mandatory
- Each rabbit hole should consume 20-40 minutes before being ruled out
- At least one rabbit hole should be a near-miss that fails for a concrete technical reason

### Story
- Team or production-like environment with multiple personas
- Vulnerabilities must come from believable operational trade-offs (speed, legacy compatibility, incomplete hardening)
- Every chain step must be narratively justified and logically connected

---

## Expected Solver Experience

```
1. Enumerate external attack surface and fingerprint stack
2. Analyze app behavior and identify nuanced initial vulnerability
3. Gain foothold and perform internal service/process enumeration
4. Pivot through trust boundary using discovered artifacts
5. Deeply enumerate target user context and automation paths
6. Chain privesc conditions to obtain root
```

Total time: **6-14 hours** for the target audience.

---

## Example Architectures

### Pattern A: Access Control Flaw → Internal API Pivot → Systemd Abuse
- Broken access control in web app exposes deployment endpoint → foothold as app user
- Internal API token from runtime metadata enables pivot to ops service
- Ops user can edit environment file for root-owned systemd service restart path → root

### Pattern B: SSRF → Service Credential Pivot → CI/CD Abuse
- SSRF reaches internal admin service with weak trust model
- Admin service reveals scoped token that grants repository runner control
- Runner executes privileged deployment job with writable script path → root

### Pattern C: SSTI → Hash Extraction → Automation Misconfiguration
- SSTI leads to shell and local data access
- Extract password hash and crack with common wordlist
- Cracked credentials unlock user with write access to root-run automation asset → root

---

## Checklist for AI Agent

Before finalizing a Hard scenario, verify:

- [ ] The attack chain has 4-5 distinct steps
- [ ] Any CVE used is from 2025 or 2026 (`CVE-2025-*` or `CVE-2026-*`)
- [ ] At least one lateral movement pivot is included
- [ ] At least one step requires chaining multiple discoveries
- [ ] Vulnerability classes are varied across the intended chain
- [ ] Two rabbit holes are included and technically defensible
- [ ] No plaintext credentials are handed over via `.env`, backup, note, or hint files
- [ ] The chain remains logically connected from first foothold to root
- [ ] No `sudo -l` giveaway privesc
- [ ] OS is Ubuntu 26.04 unless an older version is specifically justified
