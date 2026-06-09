# CTF Machine: Farewell

## Overview

**Name:** Farewell
**Difficulty:** Medium
**OS:** Windows 10
**Domain:** `farewell.ms`
**Theme:** Ed-tech startup with exposed Nginx UI and internal Chamilo LMS — chained CVEs for initial access and privilege escalation

**Story:** Farewell Academy is an ed-tech startup with a single Windows 10 machine running Chamilo LMS on port 80 (localhost only) and Nginx UI on port 17892 (accidentally exposed). A developer runs the entire stack. SSH is also open for remote administration.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web (80) | Apache/XAMPP — Chamilo LMS v1.11.36 (bound to localhost) |
| Web (17892) | Nginx UI v2.3.1 (exposed — attack surface) |
| SSH | Open for remote administration |
| Open Ports | 17892 (Nginx UI), 22 (SSH), 80 (localhost only) |
| App runs as | `nui-svc` (Nginx UI), `developer` (XAMPP Apache — local Administrators) |

---

## Attack Chain

### 1 → Nginx UI CVE Chain — Anonymous → nui-svc

Nginx UI v2.3.1 has two chained vulnerabilities:

- **CVE-2026-27944** (CVSS 9.8): `/api/backup` requires no authentication. Returns encrypted backup with decryption key in `X-Backup-Security` header. Decrypting reveals `app.ini` with `JwtSecret`.
- **CVE-2026-33032** (CVSS 9.8): `/mcp_message` endpoint missing `AuthRequired()` middleware — unauthenticated MCP access.

Forge admin JWT (HS256) with leaked `JwtSecret`. Use built-in Terminal feature → reverse shell as `nui-svc`.

🚩 **user.txt** → `C:\Users\nui-svc\Desktop\user.txt`

---

### 2 → Enumeration & Pivot — nui-svc → Internal Chamilo Access

From `nui-svc` shell: `netstat -ano` reveals port 80 on 127.0.0.1 (Apache/XAMPP). Chamilo LMS installed at `C:\xampp\htdocs\chamilo\`. Extract MariaDB creds from `configuration.php`, crack bcrypt admin hash with rockyou.txt. Port forward with chisel to expose Chamilo.

---

### 3 → Chamilo LMS CVE-2026-32892 — nui-svc → developer (Administrator)

Chamilo LMS v1.11.36: OS command injection in `fileManage.lib.php` `move()` function. The `move_to` POST parameter passed to `exec()` filtered only with `Security::remove_XSS()` (no `escapeshellarg()`). Shell metacharacters (`&`, `|`) bypass filtering.

**Exploitation:**
Login as Chamilo admin (Teacher role) → document manager → intercept move request → inject PowerShell payload via `move_to` parameter using `&` operator. XAMPP runs as `developer` who is a local Administrator.

🚩 **root.txt** → `C:\Users\developer\Desktop\root.txt`

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\nui-svc\Desktop\user.txt` | nui-svc |
| root.txt | `C:\Users\developer\Desktop\root.txt` | developer (Administrator) |

---

## Full Chain Diagram

```
Anonymous
    │  Nginx UI 17892 → CVE-2026-27944 backup leak → JwtSecret
    │  Forge admin JWT → Terminal → reverse shell
    ▼
nui-svc  →  user.txt
    │  netstat → port 80 localhost (Chamilo LMS)
    │  configuration.php → DB creds → bcrypt crack
    │  chisel port forward → Chamilo admin login
    │  CVE-2026-32892 → command injection in move_to
    │  Shell as developer (local Administrator)
    ▼
developer (Admin)  →  root.txt
```
