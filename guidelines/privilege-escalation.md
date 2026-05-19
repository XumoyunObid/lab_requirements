# Privilege Escalation — Real-World Techniques by Difficulty

Privilege escalation design is the most critical aspect of machine quality. Every privesc path must reflect what a penetration tester would encounter in a real corporate environment.

---

## Banned Techniques (All Levels)

These techniques are **strictly prohibited** as privesc vectors:

| Technique | Reason |
|-----------|--------|
| `sudo -l` with GTFOBins | No real sysadmin puts `NOPASSWD: /usr/bin/vim` in sudoers |
| SUID `find`/`vim`/`less`/`nmap` | No admin sets SUID on standard utilities |
| Writable `/etc/passwd` or `/etc/shadow` | Impossible on modern Linux without prior root access |
| World-writable cron script | Real cron jobs run `root:root 0755` scripts |
| Kernel exploit as the **only** path | "Run exploit, get root" teaches nothing |
| Simple PATH hijacking | `PATH=/tmp:$PATH` binary replacement is unrealistic |
| `LD_PRELOAD` as sole vector | `env_keep += LD_PRELOAD` does not exist in real sudoers |

---

## What Real-World Privesc Looks Like

In real corporate penetration tests, privilege escalation typically involves:

- **Cleartext credentials** — config files, `.bash_history`, `.env`, database backups
- **Service account misconfiguration** — services running with excessive privileges
- **Unpatched kernel/software CVE** — systems behind on patching
- **Scheduled task abuse** — root cron jobs calling exploitable scripts (via dependency hijack, NOT world-writable scripts)
- **Internal service exploitation** — localhost services running with elevated privileges
- **Container/Docker misconfiguration** — mounted socket, privileged mode, `cap_sys_admin`
- **AD-specific** — Kerberoasting, delegation abuse, ACL misconfiguration

---

## Linux Privilege Escalation — Approved Techniques

### Easy

| Technique | Description |
|-----------|-------------|
| Cleartext credentials in config files | `/opt/app/.env`, `/var/www/html/config.php`, `database.yml` — password reuse |
| Misconfigured service running as root | Custom app running as root on local port — exploitable |
| Readable backup/database dump | `/var/backups/db.sql` or `.bak` file with root password hash → crack |
| Docker group membership | User in `docker` group → `docker run -v /:/mnt` escape |

### Medium

| Technique | Description |
|-----------|-------------|
| Custom SUID/capability binary | SUID binary NOT in GTFOBins — requires reverse engineering (Ghidra) |
| Abusing systemd timer/service | User-writable `ExecStart` path or writable `.service` file |
| Credential in process memory / `/proc` | Running process cmdline or environ contains password |
| NFS `no_root_squash` exploitation | NFS share mountable as root → create SUID binary |
| Internal web application running as root | Localhost admin panel in root process — exploit for root |
| Abusing logrotate / mail / at | Logrotate race condition, writable at job — real CVEs |

### Hard

| Technique | Description |
|-----------|-------------|
| Custom application buffer overflow | Local SUID binary with stack/heap overflow — write exploit |
| Race condition exploitation | TOCTOU bug, symlink race — real kernel or application vulnerability |
| Container escape | Docker privileged, mounted `/var/run/docker.sock`, `cap_sys_admin` + `nsenter` |
| Kubernetes pod → node escalation | ServiceAccount token for node escape — real cloud environment |
| Abusing custom PKI/certificate infrastructure | Internal CA misconfiguration — certificate forgery |
| Process injection via ptrace/gdb | Attach to running process — credential or session theft |
| Kernel CVE exploitation (unpatched) | DirtyPipe, DirtyCow, GameOver(lay) — student must understand the mechanism |

### Insane

| Technique | Description |
|-----------|-------------|
| Kernel exploitation with custom exploit development | ret2usr, KASLR/SMEP/SMAP bypass — student writes the exploit |
| Heap exploitation in userland binary | tcache poisoning, fastbin dup, house of force |
| Cryptographic attack | Timing side-channel, padding oracle, weak custom crypto |
| Complex AD ACL chain (5+ steps) | GenericAll → WriteDACL → AddMember → DCSync |
| Hypervisor escape (simulation) | QEMU/KVM CVE-based — research-level exploitation |

---

## Windows Privilege Escalation — Approved Techniques

### Easy

| Technique | Description |
|-----------|-------------|
| SeImpersonatePrivilege | Service account — PrintSpoofer, GodPotato (real for IIS/MSSQL accounts) |

### Medium

| Technique | Description |
|-----------|-------------|
| DLL Hijacking via writable PATH directory | Service loads DLL from writable directory — real corporate misconfig |
| Token impersonation | SeImpersonatePrivilege + named pipe or potato variant |

### Hard

| Technique | Description |
|-----------|-------------|
| ADCS ESC1-ESC8 | Certificate template misconfiguration — widespread in real AD environments |
| UAC bypass + custom payload | Fodhelper, CMSTP, eventvwr bypass — real red team technique |

---

## Privesc Design Rules

1. Privesc **must** be based on a real corporate environment scenario
2. Student must learn a new **technique**, not just run an exploit
3. Enumeration tools (linpeas, winpeas, BloodHound) should help discovery, but the solution must not depend solely on them
4. Credential hunting is the basis of 60% of real privesc — config parsing and file reading are real skills
5. At every privesc step, the student must be able to answer: "Why did this work?"
6. Multi-step privesc (user → service → root) is recommended for Medium+ difficulty
