# Easy Machine Template — Constraints & Expectations

## Difficulty Profile

An **Easy** CTF machine is designed for beginners and intermediate players. It tests fundamental offensive security skills with well-documented vulnerabilities.

---

## Constraints

### Attack Chain
- **Exactly 2-3 steps:** Enumeration → Initial Access → Privilege Escalation
- Each step uses a **single, well-known technique**
- No multi-step exploitation within a single phase
- The solver should spend most time on enumeration, not exploitation

### Initial Access
- Must use a **publicly known CVE** with an existing PoC tool, OR a **common web vulnerability** (SQLi, command injection, file upload)
- The vulnerability must be in the **first service** the solver interacts with
- No authentication required to reach the vulnerable endpoint
- The PoC should work with minimal modification (change target IP/URL only)

### Privilege Escalation
- Must be discoverable through **standard enumeration** (`ps aux`, `ss -tlnp`, `find / -perm -4000`, `ls -la`, config files)
- Should exploit a **single misconfiguration** — not a chain of issues
- Must be achievable within 15 minutes of enumeration for an experienced player
- Must relate to the machine's technology stack (not a random misconfiguration)

### Rabbit Holes
- **0-1 rabbit holes**, optional
- If present, must be very shallow (solver realizes it's a dead end within minutes)

### Story
- Simple scenario: solo developer, small business, personal project
- One or two technology choices that introduce the vulnerabilities
- The "mistake" should be obvious in hindsight (never updated, ran as root, default config)

---

## Expected Solver Experience

```
1. nmap scan → identify web service and SSH
2. Web enumeration → identify framework/CMS and version
3. Research vulnerability → find CVE and PoC
4. Run PoC → get shell as application user
5. Enumerate system → find misconfigured service/process
6. Exploit misconfiguration → get root
```

Total time: **1-3 hours** for the target audience.

---

## Example Architectures

### Pattern A: Vulnerable Web Framework + Process Manager Abuse
- Nginx → App framework (known CVE) → shell as app user
- Process manager (PM2/supervisord) running as root with writable config → root

### Pattern B: Web App Vulnerability + Cron Job Abuse
- Apache → CMS with known vuln → shell as www-data
- Cron job running as root calls a script writable by www-data → root

### Pattern C: Default Credentials + Service Misconfiguration
- Web app with default admin credentials → authenticated RCE → shell
- Database running as root with UDF/file write capability → root

---

## Checklist for AI Agent

Before finalizing an Easy scenario, verify:

- [ ] The initial CVE/vulnerability has a public PoC that works out of the box
- [ ] Any CVE used is from 2025 or 2026 (`CVE-2025-*` or `CVE-2026-*`)
- [ ] The technology stack is common and well-documented
- [ ] The privesc vector is discoverable through standard `linpeas`/`LinEnum` enumeration
- [ ] The story explains both the initial vuln and the privesc naturally
- [ ] No step requires obscure knowledge or tools
- [ ] The entire chain can be solved without hints in under 3 hours
- [ ] `sudo -l` is NOT the privesc vector
- [ ] No plaintext credentials are handed over via `.env`, backup, note, or hint files
- [ ] Passwords requiring cracking are found in common wordlists (rockyou.txt, etc.)
- [ ] OS is Ubuntu 26.04 unless an older version is specifically justified
