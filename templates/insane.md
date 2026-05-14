# Insane Machine Template — Constraints & Expectations

## Difficulty Profile

An **Insane** CTF machine is designed for advanced players. It requires deep technical knowledge, custom exploit development, creative thinking, and thorough enumeration. Multiple false paths exist at every stage.

---

## Constraints

### Attack Chain
- **4-6 steps:** Enumeration → Initial Access → Lateral Movement(s) → Privilege Escalation
- Must include at least **one lateral movement** (pivot between users, services, or network segments)
- At least one step must require **custom exploit development** or significant PoC modification
- Steps must combine **multiple vulnerability classes** (e.g., web + crypto + binary + config)
- The "intended path" should not be obvious even to experienced CTF players

### Initial Access
- May use a **recent or obscure CVE** with limited public tooling
- May require **multi-step authentication bypass** (e.g., OAuth flaw, SAML manipulation, session fixation chain)
- May require **custom vulnerability exploitation** (business logic flaw, race condition, prototype pollution chain)
- Blind/out-of-band exploitation is appropriate (blind XXE + OOB, blind SSRF + DNS rebinding)
- The solver may need to write their own exploit or significantly modify existing tools

### Lateral Movement (Required, 1-2 pivots)
- Each pivot must cross a meaningful **trust boundary**
- Pivots may involve:
  - Compromising internal microservices through service-to-service trust
  - Extracting credentials from process memory, environment variables, or sealed secrets
  - Abusing API tokens or service accounts with cross-service access
  - Network pivoting to an internal host via the compromised foothold

### Privilege Escalation
- Must require **research and creativity** — not a standard technique
- May involve:
  - Custom binary exploitation (with source code or reversing required)
  - Cryptographic weakness exploitation (weak key derivation, predictable tokens)
  - Multi-step privilege chains (combining 2+ low-severity issues)
  - Supply chain attacks (writable package paths, LD_PRELOAD with writable library directories)
  - Kernel exploitation ONLY if the story involves a deliberately unpatched legacy/embedded system
- The solver should need to understand the *why*, not just the *how*

### Rabbit Holes
- **2-3 rabbit holes**, mandatory
- Each should be deeply convincing — a solver might spend 30-60 minutes before ruling it out
- At least one rabbit hole should be a **near-miss** (similar to the real path but with one crucial difference)
- Rabbit holes should teach advanced concepts

### Story
- Complex environment: enterprise, multi-service architecture, DevOps pipeline, or research lab
- Multiple personas with different roles and access levels
- Vulnerabilities arise from **architectural decisions**, not just individual mistakes
- The system should look "properly secured" on the surface

---

## Expected Solver Experience

```
1. Thorough external enumeration → identify attack surface
2. Deep application analysis → understand business logic and data flows
3. Identify and exploit initial vulnerability (may require custom tooling)
4. Enumerate internal environment from foothold
5. Discover and exploit internal service (lateral movement)
6. Pivot to higher-privilege user/service
7. Deep system enumeration → identify non-obvious privesc vector
8. Research and exploit privesc (may require custom development)
9. Root
```

Total time: **8-24+ hours** for the target audience.

---

## Example Architectures

### Pattern A: OAuth Bypass → Microservice SSRF → Binary Exploit
- Custom OAuth implementation with token validation flaw → admin API access
- Admin API can reach internal service → SSRF to internal metrics service
- Metrics service leaks process environment → credentials for `ops` user
- `ops` user can restart a custom monitoring daemon (SUID) with a buffer overflow → root

### Pattern B: Blind XXE + OOB → Database Pivot → Ansible Vault Crack → Root
- Web app accepts XML → blind XXE with out-of-band data exfiltration
- Exfiltrate database config → access PostgreSQL with COPY TO for file read
- Read Ansible vault file → crack weak vault password
- Vault contains SSH key for `devops` user → writable CI/CD pipeline → root

### Pattern C: Prototype Pollution → RCE → Container Escape → Host Root
- Node.js API with prototype pollution in merge endpoint
- Chain pollution to RCE via child_process gadget → shell in container
- Container has access to Docker socket (mounted for health checks)
- Create privileged container with host filesystem mounted → root on host

---

## Checklist for AI Agent

Before finalizing an Insane scenario, verify:

- [ ] The attack chain has 4-6 distinct steps
- [ ] At least one step requires custom exploit development or significant PoC modification
- [ ] At least one lateral movement pivot is included
- [ ] The privesc is non-obvious and requires research
- [ ] 2-3 rabbit holes are included and are deeply convincing
- [ ] Each rabbit hole teaches an advanced concept
- [ ] The story is complex enough to justify the architecture
- [ ] All vulnerabilities are realistically motivated by the story
- [ ] The chain would challenge an experienced CTF player for 8+ hours
- [ ] No `sudo -l` giveaway privesc
- [ ] No plaintext credential files used as hints
- [ ] Passwords requiring cracking are found in common wordlists (rockyou.txt, etc.)
- [ ] OS is Ubuntu 26.04 unless an older version is specifically justified
- [ ] Multiple false paths exist at each decision point
- [ ] The "intended path" is not the most obvious approach
