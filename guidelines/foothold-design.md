# Foothold Design — Realistic Scenarios

The foothold is the first point of entry into a machine. It is the most time-consuming phase of a real penetration test, and the design must reflect that reality.

---

## General Rules

- Foothold must be discoverable through standard enumeration — `nmap` + web fuzzing should yield results within 30 minutes
- Public CVEs are acceptable at Easy/Medium level and may be used directly
- At Hard+ level, public exploits should require modification (offset, payload, bypass adjustments)
- Brute-force must **never** be the sole path — but credential stuffing from a leaked list is realistic
- Default credentials are acceptable only in **real software** (Tomcat, Jenkins, Grafana, phpMyAdmin)
- Custom web application vulnerabilities must map to **OWASP Top 10**

---

## Web Application Footholds (Most Common)

| Technique | Description |
|-----------|-------------|
| **SQL Injection → RCE** | SQLi in a real webapp leading to shell via `INTO OUTFILE`, `xp_cmdshell`, or stacked queries |
| **Insecure File Upload** | Real CMS or custom upload — extension bypass, content-type manipulation |
| **SSTI** | Server-Side Template Injection in Jinja2, Twig, Freemarker, or other real template engines |
| **Insecure Deserialization** | Java (ysoserial), PHP (unserialize), Python (pickle) in real applications |
| **Authentication Bypass** | JWT manipulation, broken auth logic, default credentials in real software |
| **SSRF → Internal Service** | SSRF to reach internal metadata, Redis, or other internal services |
| **Exposed Git/SVN Repository** | `.git` directory exposed → source code recovery → hardcoded credentials |
| **API Misconfiguration** | GraphQL introspection, mass assignment, IDOR — real API vulnerabilities |
| **Known CVE in Real Software** | CVEs in real, widely-deployed software with public PoC |

---

## Network Service Footholds

| Technique | Description |
|-----------|-------------|
| **SMB/NFS Share + Credential Leakage** | Anonymous or guest access to shares containing config files with passwords |
| **FTP + Writable Web Directory** | Anonymous FTP with write access to web root — webshell upload |
| **SNMP v1/v2 Community String** | Public/private community string leaking running processes or credentials |
| **LDAP Anonymous Bind** | AD environment — enumerate users, groups, description fields with passwords |
| **Old Service Version + Public CVE** | ProFTPd, vsftpd, OpenSMTPD — real CVE exploitation |
| **Redis/MongoDB/Elasticsearch Exposed** | Unauthenticated internal database — data extraction or RCE |

---

## Foothold by Difficulty

### Easy
- Single well-known CVE with public PoC, or common web vulnerability
- Vulnerability is in the first service the solver interacts with
- No authentication required to reach the vulnerable endpoint

### Medium
- CVE requiring configuration knowledge to exploit
- May require authenticated access first (registration, default creds in real software)
- Custom vulnerabilities (XXE, SSTI, SSRF, deserialization) preferred

### Hard
- Vulnerability requires understanding request flow, auth logic, or deployment context
- Public exploit must be modified to work (offsets, payloads, bypass techniques)
- Multi-step initial access is expected

### Insane
- Recent or obscure CVE with limited public tooling
- Custom exploit development or significant PoC modification required
- Multi-step authentication bypass or blind/out-of-band exploitation
