# Medium Machine Template — Constraints & Expectations

## Difficulty Profile

A **Medium** CTF machine challenges experienced beginners and intermediate players. It introduces vulnerability chaining, lateral movement, and requires deeper understanding of the technology stack.

---

## Constraints

### Attack Chain
- **Exactly 3-4 steps:** Enumeration → Initial Access → [Lateral Movement] → Privilege Escalation
- At least one step must require **chaining two pieces of information** (e.g., finding a key via LFI that enables SSH access)
- Lateral movement between users or services is strongly encouraged
- Each step should use a **different attack technique**

### Initial Access
- May use a CVE that requires **specific configuration knowledge** to exploit
- May require **authenticated access** first (register an account, find default creds in app-level config)
- Custom vulnerabilities (XXE, SSTI, SSRF, deserialization) are preferred over simple CVEs
- The solver must understand the vulnerability class, not just run a tool

### Lateral Movement (Recommended)
- Move between users or services on the same machine
- Discovery should come from **enumerating the access gained** in the previous step
- Common patterns:
  - Internal-only API/service exploitable from the foothold
  - Credentials found in application config, database, or process environment
  - SSH key exfiltration via web vulnerability

### Privilege Escalation
- Must require **understanding** of the tool/service being abused
- Infrastructure-as-Code tools (Ansible, Puppet, Chef), CI/CD pipelines, or custom automation are preferred
- The solver must understand why the misconfiguration is dangerous, not just exploit it mechanically
- Multiple enumeration steps may be needed to identify the vector

### Rabbit Holes
- **1-2 rabbit holes**, mandatory
- Each must be convincing enough that a solver spends 15-30 minutes investigating
- Must have a clear technical reason for being a dead end (not just "it doesn't work")
- Should teach something about the technology stack

### Story
- Small team or startup scenario with multiple roles (developer, sysadmin, DevOps)
- Multiple technology choices that interact in insecure ways
- The vulnerabilities arise from **team miscommunication** or **rushed deployment**
- At least one "it works so don't touch it" situation

---

## Expected Solver Experience

```
1. nmap scan → identify web service and SSH
2. Web enumeration → understand the application's functionality
3. Identify vulnerability class (XXE, SSTI, SSRF, etc.)
4. Craft exploit → get initial foothold or exfiltrate data
5. Enumerate internal services/configs → discover lateral movement path
6. Pivot to new user/service using discovered credentials/keys
7. Enumerate new user's access → find automation/deployment misconfiguration
8. Exploit misconfiguration → get root
```

Total time: **3-8 hours** for the target audience.

---

## Example Architectures

### Pattern A: SSTI → Internal LFI → Ansible Abuse
- Web app with template injection → shell as app user
- Internal API with LFI → read SSH key for developer user
- Developer has write access to Ansible playbooks run by root → root

### Pattern B: XXE → SSH Key Exfil → APT Hook Abuse
- XML-based registration with XXE → exfiltrate SSH private key
- SSH as user → find cron running `apt-get update` as root
- APT hook script is world-writable → root

### Pattern C: Auth Bypass → Database Dump → Container Escape
- JWT with weak secret → admin access to API
- API reveals database credentials → dump user table with SSH password
- User has Docker group membership → container escape to root

---

## Checklist for AI Agent

Before finalizing a Medium scenario, verify:

- [ ] The attack chain has 3-4 distinct steps
- [ ] At least one step requires understanding the vulnerability, not just running a tool
- [ ] Lateral movement (if present) uses a different technique than initial access
- [ ] The privesc requires understanding of the automation/deployment tool
- [ ] 1-2 rabbit holes are included and are technically convincing
- [ ] Each step logically leads to the next through enumeration
- [ ] The story explains all vulnerabilities and misconfigurations naturally
- [ ] No `sudo -l` giveaway privesc
- [ ] No plaintext credential files used as hints
- [ ] The chain would survive review by an experienced penetration tester
