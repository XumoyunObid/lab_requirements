# CTF Machine: MedVault

## Overview

**Name:** MedVault
**Difficulty:** Medium
**OS:** Windows
**Domain:** `medvault.ms`
**Theme:** Medical records portal with Jinja2 SSTI leading to RCE, privilege escalation via SeImpersonatePrivilege + GodPotato

**Story:** MedVault is a medical records management system (patient cards, prescriptions, lab results) built with Python Flask. The patient name field is passed directly to `render_template_string()` without sanitization, enabling Server-Side Template Injection. The `medvault-svc` service account has SeImpersonatePrivilege, allowing token impersonation to SYSTEM via GodPotato.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | Python Flask — medical records management (port 8080) |
| Template Engine | Jinja2 (render_template_string — SSTI sink) |
| Open Ports | 8080 (Flask) |
| App runs as | `medvault-svc` (service account, SeImpersonatePrivilege) |

---

## Attack Chain

### 1 → Jinja2 SSTI — Anonymous/Authenticated → medvault-svc

The patient name field is passed directly to `render_template_string()`. Confirm SSTI with `{{7*7}}` → 49. Escalate to RCE via Jinja2 sandbox escape.

**Exploitation:**
```
{{7*7}} → 49  (SSTI confirmed)
{{config.__class__.__init__.__globals__['os'].popen('whoami').read()}} → medvault-svc
```

Reverse shell as `medvault-svc`.

**CWE:** CWE-1336 (Improper Neutralization of Special Elements Used in a Template Engine)

🚩 **user.txt** → `C:\Users\medvault-svc\Desktop\user.txt`

---

### 2 → SeImpersonatePrivilege + GodPotato — medvault-svc → SYSTEM

`whoami /priv` shows SeImpersonatePrivilege enabled on the service account. Transfer GodPotato binary, execute to impersonate SYSTEM token via local NTLM relay.

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\medvault-svc\Desktop\user.txt` | medvault-svc |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator / SYSTEM |

---

## Full Chain Diagram

```
Anonymous
    │  Patient name field → {{7*7}} → SSTI confirmed
    │  Jinja2 sandbox escape → RCE
    │  Reverse shell as medvault-svc
    ▼
medvault-svc  →  user.txt
    │  whoami /priv → SeImpersonatePrivilege
    │  GodPotato → SYSTEM token impersonation
    ▼
SYSTEM  →  root.txt
```
