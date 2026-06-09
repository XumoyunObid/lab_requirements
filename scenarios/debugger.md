# CTF Machine: Debugger

## Overview

**Name:** Debugger
**Difficulty:** Medium
**OS:** Windows 10 Pro
**Domain:** `debugger.ms`
**Theme:** Corporate website with internal Joomla file-sharing subdomain — .NET binary reverse engineering reveals MSSQL credentials, JCE Editor CVE for privilege escalation

**Story:** A corporate website conceals an internal subdomain (`files.debugger.ms`) used by QA testers for file sharing, powered by Joomla. Directory enumeration reveals an exposed uploads folder containing a .NET database monitoring tool with hardcoded MSSQL credentials. The Joomla installation has JCE Editor < 2.9.99.5, vulnerable to CVE-2026-48907, but the endpoints are restricted to localhost via IIS URL Authorization — exploitable only after gaining a foothold.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web (80/443) | IIS — Main corporate site (HTTPS, secure) + `files.debugger.ms` vhost (Joomla) |
| Database | MSSQL (port 1433) — `AssetDB` |
| Joomla | JCE Editor < 2.9.99.5 (CVE-2026-48907) — localhost only |
| Open Ports | 80, 443, 1433 (MSSQL), 5985 (WinRM), 445 (SMB) |
| IIS AppPool | `files.debugger.ms` runs as Administrator (misconfiguration) |

---

## Attack Chain

### 1 → Subdomain + Directory Enumeration — Anonymous → Binary Discovery

Nmap finds 80, 443, 445, 1433, 5985. Main site is hardened. Vhost fuzzing discovers `files.debugger.ms` running Joomla. Directory brute-force finds `/uploads/` with directory listing enabled containing `DBHealthMonitor.exe`.

---

### 2 → .NET Reverse Engineering — Binary → MSSQL Credentials

Decompile `DBHealthMonitor.exe` in dnSpy/ILSpy. `DatabaseHelper.BuildConnectionString()` contains hardcoded credentials: `svc_dbmon / Db_M0nit0r!2025` targeting `AssetDB` on `debugger.ms`.

---

### 3 → MSSQL xp_cmdshell — Anonymous → svc_mssql

Connect with impacket-mssqlclient, enable `xp_cmdshell`, execute commands as `svc_mssql`.

**Exploitation:**
```bash
impacket-mssqlclient svc_dbmon:'Db_M0nit0r!2025'@debugger.ms
SQL> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
SQL> EXEC xp_cmdshell 'whoami';
# debugger\svc_mssql
```

🚩 **user.txt** → `C:\Users\svc_mssql\Desktop\user.txt`

---

### 4 → CVE-2026-48907 JCE Editor — svc_mssql → Administrator

From the foothold shell, enumerate web root → find Joomla with JCE Editor < 2.9.99.5. JCE endpoints are localhost-only (IIS URL Authorization). Exploit CVE-2026-48907: create malicious editor profile allowing PHP uploads, upload PHP shell, trigger it. IIS AppPool runs as Administrator.

**CVE:** CVE-2026-48907 — JCE Editor Insufficient Access Controls (CVSS 10.0)

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Rabbit Holes

### Rabbit Hole 1: SMB (445)
**What the solver finds:** SMB share enumeration with guest session.
**Why it looks promising:** Anonymous SMB access could leak files.
**Why it's a dead end:** All shares require valid domain credentials — no anonymous read/write.

### Rabbit Hole 2: JCE from External
**What the solver finds:** JCE is installed on the Joomla instance.
**Why it looks promising:** CVE-2026-48907 is a CVSS 10.0 unauthenticated RCE.
**Why it's a dead end:** Editor endpoints return 403 from external IPs — requires local access after foothold.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\svc_mssql\Desktop\user.txt` | svc_mssql |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator |

---

## Full Chain Diagram

```
Anonymous
    │  Vhost fuzzing → files.debugger.ms (Joomla)
    │  Directory brute-force → /uploads/DBHealthMonitor.exe
    │  dnSpy decompile → MSSQL creds (svc_dbmon)
    │  impacket-mssqlclient → xp_cmdshell → shell
    ▼
svc_mssql  →  user.txt
    │  Enumerate web root → Joomla + JCE < 2.9.99.5
    │  CVE-2026-48907 from localhost → PHP upload
    │  IIS AppPool = Administrator
    ▼
Administrator  →  root.txt
```
