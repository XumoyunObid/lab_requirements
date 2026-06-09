# CTF Machine: Badge

## Overview

**Name:** Badge
**Difficulty:** Medium
**OS:** Windows
**Domain:** `badge.ms`
**Theme:** HID-badge firmware update portal with weak signature verification and SCCM cache MSI hijacking

**Story:** BadgeWare Inc. is an ID-badge manufacturer that lets customers flash custom firmware to HID Prox cards through a web portal. The `/firmware` endpoint accepts `.bin` files and verifies a detached PKCS#7 signature, but only checks that the blob has any valid signature from the corporate CA — not the intended OID. The box is an SCCM client with mis-ACL'd cache folders, allowing MSI hijacking for SYSTEM execution.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | Firmware update portal — `/firmware` endpoint |
| Signature Check | PKCS#7 verification (CA-signed, no OID check) |
| Scheduled Task | `BadgeFlashTask` — runs every 5 min, executes `UpdateService.exe` |
| SCCM | Client with cached packages in `C:\Windows\ccmcache\` |
| App runs as | `NT AUTHORITY\LocalService` |

---

## Attack Chain

### 1 → Weak Signature Verification — Anonymous → LocalService

The portal checks only that the uploaded blob has any valid signature from the corporate CA — no OID verification. Grab a legit driver catalog file (.cat) signed by the same CA, append a malicious DLL as overlay data, rename to `.bin`. Upload succeeds.

The service writes the blob to `C:\ProgramData\BadgeWare\updates\`. A scheduled task (`BadgeFlashTask`) runs every 5 minutes, executing `UpdateService.exe` which `LoadLibrary`s the first PE in that directory.

**Exploitation:**
Rogue DLL spawns a reverse shell as `LocalService`.

🚩 **user.txt** → `C:\Users\LocalService\Desktop\user.txt` (or equivalent)

---

### 2 → SCCM Cache MSI Hijack — LocalService → SYSTEM

The box is an SCCM client. Cached packages live in `C:\Windows\ccmcache\`. `LocalService` has write access to the cache folder (mis-ACL'd by previous admin). An upcoming mandatory deployment references `BadgeAgent.msi` running as SYSTEM.

**Exploitation:**
Pre-place a malicious MSI with the same name + correct product GUID. At next SCCM policy refresh, the fake MSI executes with SYSTEM rights — custom action spawns reverse shell.

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | LocalService profile | LocalService |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | SYSTEM |

---

## Full Chain Diagram

```
Anonymous
    │  Grab legit .cat file signed by corporate CA
    │  Append malicious DLL as overlay → rename to .bin
    │  Upload to /firmware endpoint (OID not checked)
    │  BadgeFlashTask LoadLibrarys rogue DLL
    ▼
LocalService  →  user.txt
    │  SCCM client → ccmcache writable
    │  Replace BadgeAgent.msi with malicious MSI
    │  SCCM policy refresh → SYSTEM execution
    ▼
SYSTEM  →  root.txt
```
