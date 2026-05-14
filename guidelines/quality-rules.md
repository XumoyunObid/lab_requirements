# CTF Scenario Quality Rules

These rules are **mandatory** for every scenario the AI agent generates. They ensure realism, logical coherence, and educational value.

---

## 1. Story-Driven Vulnerabilities

Every vulnerability in the attack chain **must** be justified by the scenario's story.

- ✅ A solo developer who never updated his Next.js app → known CVE exists
- ✅ A sysadmin who fat-fingered `chmod 757` instead of `755` → writable script
- ❌ A random `backup.txt` with root credentials in `/tmp`
- ❌ An unexplained SUID binary that gives root

Ask: *"Would a real person in this role realistically make this mistake?"* If the answer is no, pick a different vulnerability.

---

## 2. No Contrived Hints

The solver must discover information through **realistic enumeration**, not planted breadcrumbs.

### Forbidden patterns:
- `notes.txt`, `todo.txt`, `credentials.txt` with passwords or usernames
- Backup files (`*.bak`, `*.old`, `*.backup`) containing credentials
- Developer notes or README files with hardcoded passwords
- Comments in config files saying `# TODO: remove this password`
- `.bash_history` conveniently containing the exact command needed
- Readable `/etc/shadow` without narrative justification
- Desktop files with hints like `password_reminder.txt`
- Any file whose sole purpose is to hand the solver a credential

### Acceptable discovery methods:
- Reading application source code that reveals database connection strings
- Enumerating running processes (`ps aux`) to discover services
- Network scanning internal ports (`ss -tlnp`) to find internal APIs
- Reading legitimate config files (nginx, PM2, docker-compose) that reveal architecture
- Checking file permissions on directories the user owns

---

## 3. No Lazy Privilege Escalation

Privilege escalation must require **understanding** of the system, not just running `sudo -l` and copying a GTFOBins command.

### Forbidden privesc patterns:
- `sudo -l` showing a binary with NOPASSWD that directly gives root shell
- Random SUID binaries that aren't part of the story
- Kernel exploits on outdated kernels (unless the story specifically justifies an old unpatched server)
- Cron jobs running as root with world-writable scripts (unless narratively justified with a specific reason)
- Writable `/etc/passwd` or `/etc/shadow`

### Acceptable privesc patterns:
- Abusing a process manager (PM2, systemd, supervisord) that runs as root due to documented setup procedures
- Exploiting Infrastructure-as-Code tools (Ansible, Puppet, Chef) with writable playbooks
- Leveraging CI/CD pipelines (Jenkins, GitLab Runner) with excessive permissions
- Abusing container escape in Docker/Kubernetes when the app is intentionally containerized
- Exploiting custom internal services that run as root by design
- Writable APT/dpkg hooks when a cron runs package management as root (with narrative reason)
- Abusing database admin access to write files or execute commands

---

## 4. Logical Chain Connectivity

Each step in the attack chain must **naturally lead** to the next. The solver should discover the next step through enumeration of what they gained in the previous step.

### Chain rules:
- Step N must provide access or information that makes Step N+1 **discoverable**
- The solver should not need external hints to find the next step
- Each step should involve a different technique or attack surface
- Lateral movement must involve a clear privilege boundary (different user, different service, different network segment)

### Example of good connectivity:
1. Web exploit → shell as `www-data`
2. `www-data` can read app config → finds database credentials
3. Database contains SSH key for `developer` user
4. `developer` has write access to ansible playbook → root via automation

### Example of bad connectivity:
1. Web exploit → shell as `www-data`
2. Random SUID binary → root (no logical connection to the web app)

---

## 5. Realistic Architecture

The machine's architecture must reflect **real-world deployments**.

### Requirements:
- Specify the OS and version
- List all open ports with the service behind each
- Describe the technology stack with version numbers
- Explain how the application is deployed and managed
- The architecture must match the story (a solo developer's setup looks different from an enterprise deployment)

### Port discipline:
- Only expose ports that the story justifies
- Standard: SSH (22), HTTP/HTTPS (80/443)
- Additional ports must be explained (e.g., "MongoDB runs on 27017 because the developer never bound it to localhost")
- Internal-only services should NOT be port-scanned from outside

---

## 6. Rabbit Holes (Medium and Insane only)

Rabbit holes are **intentional dead ends** that waste the solver's time. They must be:

- **Believable** — they should look like a real attack vector initially
- **Discoverable** — the solver should naturally encounter them during enumeration
- **Ultimately dead** — they must not lead anywhere useful
- **Educationally valuable** — they should teach the solver to verify assumptions

### Good rabbit holes:
- A login endpoint that accepts XML but has XXE protections enabled
- A KeePass database in a user's home directory with irrelevant old passwords
- An outdated kernel version that looks exploitable but the specific vulnerability is patched
- A Docker socket that exists but the user doesn't have access to the docker group

### Bad rabbit holes:
- Random misleading files with no relation to the story
- Services that serve no narrative purpose
- Intentionally broken exploits that waste time without teaching anything

---

## 7. Flag Placement

Flags are always placed at these exact locations:

| Flag | Path | Required Access |
|------|------|----------------|
| `user.txt` | `/home/<user>/user.txt` | User-level shell |
| `root.txt` | `/root/root.txt` | Root-level access |

- `<user>` is the first user the solver compromises after the initial web exploit
- The user flag should be readable only by that user (and root)
- The root flag should be readable only by root

---

## 8. Password and Credential Realism

When password cracking or brute forcing is part of the attack chain, the passwords **must** be found in well-known, commonly used wordlists:

### Acceptable wordlists:
- `rockyou.txt`
- `xato-net-10-million-passwords` (and its variants: 10k, 100k, 1M)
- `SecLists` common credential lists
- Default application credentials (e.g., `admin:admin` for a specific app)

### Forbidden credential sources:
- Backup files (`database.bak`, `config.old`, `credentials.backup`)
- Developer notes (`notes.txt`, `todo.md`, `README-internal.md`)
- Plaintext files left on disk with passwords
- `.bash_history` with credentials in command arguments

The solver must discover the password through legitimate brute-force attacks against services (SSH, web login, database) using standard penetration testing wordlists, OR find credentials in realistic application config files (`.env`, `config.py`, `wp-config.php`, `application.yml`).

---

## 9. Realistic Vulnerability Selection

CTF labs should be as **realistic as possible**, using the latest and most commonly exploited vulnerabilities. Prioritize:

- **OWASP Top 10** vulnerabilities (Injection, Broken Access Control, SSRF, Security Misconfiguration, etc.)
- **Recently disclosed CVEs** affecting popular software
- **Common real-world misconfigurations** seen in actual penetration tests
- **Vulnerability classes** that match current threat landscapes

Avoid outdated or contrived vulnerability patterns that don't reflect modern attack surfaces.

---

## 10. Default Operating System

The available Ubuntu server versions are: **24.04**, **25.04**, and **26.04**.

- **Ubuntu 26.04** is the recommended default for all scenarios
- Use an older version (24.04 or 25.04) **only** when the scenario specifically requires it (e.g., kernel-level CVEs that target older kernel versions)
- Always specify the exact OS version in the scenario's Architecture section

---

## 11. CVE and Vulnerability Selection

- Use **real CVEs** whenever possible, with the correct CVE ID and CVSS score
- The CVE must affect the **exact version** of software specified in the architecture
- For custom vulnerabilities (e.g., XXE, LFI, SSTI), cite the relevant CWE
- Specify whether a public PoC exists and link to it conceptually
- The vulnerability must be **exploitable** given the machine's configuration

See [`vulnerability-selection.md`](vulnerability-selection.md) for detailed guidance per difficulty level.
