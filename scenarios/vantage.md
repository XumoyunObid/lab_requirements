# CTF Machine: Vantage

## Overview

**Name:** Vantage
**Difficulty:** Medium
**OS:** Windows 10 Pro
**Domain:** `vantage.ms`
**Theme:** Mid-size enterprise software company running an outdated Joomla CMS on Windows with an outsourced AV solution

**Story:** Vantage Systems builds internal tooling for financial clients. Their public-facing Joomla site is managed by James Holloway, a systems engineer with poor security habits whose personal details are publicly listed on the staff page. A previous contractor built a custom Joomla component (`com_vantage`) with raw SQL queries, never audited it, and left. The outsourced AV vendor added `C:\DevTools\` to the exclusion list for developer testing — the contract ended but the exclusion stayed.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | IIS + PHP → Joomla CMS (port 80/443) |
| Database | SQLite (embedded in com_vantage component) |
| Custom Component | com_vantage — staff directory with raw SQL queries |
| Open Ports | 80/443 (HTTP/HTTPS), 3389 (RDP — NLA enabled), 5985 (WinRM) |
| App runs as | `j.holloway` (Joomla admin, SeImpersonatePrivilege) |

---

## Attack Chain

### 1 → OSINT + Web Enumeration — Anonymous → Username & Custom Component Discovery

The corporate site has a `/staff.html` page listing employee names, roles, and direct email addresses. Email format reveals usernames: `j.holloway`, `s.chen`, `m.webb`. Joomla generator meta tag is visible in source. Gobuster/JoomScan discovers `/administrator/` and `/components/com_vantage/`.

- Nmap scan → IIS on 80/443, RDP on 3389, WinRM on 5985
- `/staff.html` → employee details including James Holloway's birthday, hometown, football club
- Joomla meta tag + directory enumeration → `/components/com_vantage/`

---

### 2 → SQL Injection in Custom Component — Anonymous → Hash Extraction → Cracked Credentials

The custom component at `index.php?option=com_vantage&view=staff&id=1` passes the `id` parameter into a raw SQLite query with no sanitisation. Single quote triggers a database error confirming injection.

**Exploitation:**
```
?option=com_vantage&view=staff&id=1 UNION SELECT 1,username,password,email FROM vntg_users--
```

Leaks James Holloway's salted MD5 hash. Crack with Hashcat:
```bash
hashcat -m 10 hash.txt /usr/share/wordlists/rockyou.txt
```

Password: `liverpool1` (predictable — Liverpool FC supporter per staff page). Login to `/administrator/` as `j.holloway`.

---

### 3 → Authenticated Template RCE — j.holloway (Joomla Admin) → Web Shell → User Flag

In Joomla admin panel: Extensions → Templates → Templates. Edit the active Protostar theme's `error.php`, inject a PHP web shell.

**Exploitation:**
```bash
curl "https://vantage.ms/templates/protostar/error.php?cmd=whoami"
# Output: vantage\j.holloway
```

Trigger a PowerShell reverse shell → shell as `j.holloway` → read `user.txt`.

**Why it works:** Joomla templates are PHP files editable from the admin panel. Authenticated admins can inject arbitrary PHP code.

---

### 4 → SeImpersonatePrivilege + GodPotato — j.holloway → SYSTEM

`whoami /priv` shows SeImpersonatePrivilege (granted by previous contractor for deployment scripts, never revoked). `C:\DevTools\` is AV-excluded and writable by Authenticated Users.

**Exploitation:**
```bash
# Transfer GodPotato to C:\DevTools\ (AV-excluded)
# Execute from excluded folder → NTLM relay → SYSTEM
```

**Why it works:** AV exclusion on `C:\DevTools\` allows the binary to run without detection. SeImpersonatePrivilege enables local NTLM relay to SYSTEM.

🚩 **user.txt** → `C:\Users\j.holloway\Desktop\user.txt`
🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Rabbit Holes

### Rabbit Hole 1: RDP (Port 3389)
**What the solver finds:** RDP port is open.
**Why it looks promising:** RDP with known credentials is a common attack path.
**Why it's a dead end:** NLA is enabled — external brute-force is blocked. Joomla web vector is the only viable entry.
**What it teaches:** Open ports don't always mean exploitable services.

### Rabbit Hole 2: WinRM (Port 5985)
**What the solver finds:** WinRM port is open.
**Why it looks promising:** evil-winrm with cracked credentials could provide a shell.
**Why it's a dead end:** `j.holloway` is not in Remote Management Users group.
**What it teaches:** Service access depends on group membership, not just credentials.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\j.holloway\Desktop\user.txt` | j.holloway |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator / SYSTEM |

---

## Full Chain Diagram

```
Anonymous
    │  OSINT: staff page → usernames, personal details
    │  Gobuster/JoomScan → com_vantage custom component
    │  SQLi in com_vantage → salted MD5 hash
    │  Hashcat → liverpool1
    │  Joomla admin login → template RCE
    ▼
j.holloway  →  user.txt
    │  whoami /priv → SeImpersonatePrivilege
    │  C:\DevTools\ AV exclusion
    │  GodPotato → SYSTEM
    ▼
SYSTEM  →  root.txt
```
