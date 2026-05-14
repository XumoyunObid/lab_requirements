# CTF Scenario Anti-Patterns

This document lists common mistakes in CTF machine design that the AI agent **must avoid**. Each anti-pattern includes an explanation of why it's unrealistic and what to do instead.

---

## 🚫 Anti-Pattern 1: The Helpful Note

**What it looks like:**
```
/home/user/notes.txt:
  "Remember: the database password is Passw0rd123!"
```

**Why it's bad:** No real person writes their passwords in plaintext notes on a production server. This is a lazy shortcut that breaks immersion and teaches nothing.

**What to do instead:** Place credentials in application config files where they belong — `.env` files, `config.py`, `application.yml`, `wp-config.php`. The solver finds them by reading source code, not by stumbling on notes. When password cracking is needed, use passwords from common brute-force wordlists like `rockyou.txt` or `xato-net-10-million-passwords`.

---

## 🚫 Anti-Pattern 2: The sudo -l Giveaway

**What it looks like:**
```
User dev may run the following commands:
    (ALL) NOPASSWD: /usr/bin/vim
```

**Why it's bad:** In reality, sysadmins rarely give users sudo access to editors or interpreters. This is the most overused CTF pattern and it's almost never realistic.

**What to do instead:** Use process managers, automation tools, or cron jobs that run as root and interact with files the user can modify. Think: "What root-level process would a lazy sysadmin set up that a user could abuse?"

---

## 🚫 Anti-Pattern 3: The Kernel Exploit Shortcut

**What it looks like:**
```
uname -r → 4.4.0-21-generic
Google "linux 4.4.0-21 exploit" → DirtyCow → root
```

**Why it's bad:** Kernel exploits are a brute-force approach that bypasses the machine's design. Unless the story specifically involves an unpatched legacy server, kernel exploits make the scenario feel lazy.

**What to do instead:** Design privilege escalation around application-layer misconfigurations — PM2, Docker, Ansible, Jenkins, cron + writable scripts, systemd timers, database UDFs.

---

## 🚫 Anti-Pattern 4: The Exposed Credential File

**What it looks like:**
```
/.git/ exposed on web server with credentials in commit history
/backup/db_dump.sql with user table containing bcrypt hashes
/robots.txt pointing to /admin with default credentials
```

**Why it's bad (sometimes):** These can be realistic but are massively overused. If you use one, it must be central to the story, not a random discovery.

**What to do instead:** If credentials are part of the chain, embed them in a realistic context — a MongoDB instance with no auth (because the developer followed a tutorial that didn't cover auth), a Redis instance bound to 0.0.0.0 (because the developer copied a Docker config), application config files with database connection strings. When brute forcing is required, use passwords that exist in well-known wordlists (rockyou.txt, xato-net-10-million-passwords, SecLists).

---

## 🚫 Anti-Pattern 5: The Random SUID Binary

**What it looks like:**
```
find / -perm -4000 2>/dev/null
/usr/local/bin/custom-tool (SUID root, runs system() with user input)
```

**Why it's bad:** Random custom SUID binaries don't exist in real environments. If a SUID binary is part of the chain, it must have a narrative reason for existing and for being SUID.

**What to do instead:** Use legitimate SUID binaries that are part of the story's technology stack, or use non-SUID privilege escalation vectors entirely.

---

## 🚫 Anti-Pattern 6: The Convenient .bash_history

**What it looks like:**
```
cat /home/user/.bash_history
mysql -u root -pSecretPass123
ssh admin@10.10.10.5
```

**Why it's bad:** While `.bash_history` can contain useful info in real environments, using it as the primary discovery mechanism is lazy. It's the CTF equivalent of "and then they found a sticky note."

**What to do instead:** Let the solver discover services through process enumeration (`ps aux`, `ss -tlnp`), config files, or systemd unit files. These are realistic discovery methods.

---

## 🚫 Anti-Pattern 7: The Disconnected Chain

**What it looks like:**
```
Step 1: SQL injection on web app → shell as www-data
Step 2: (completely unrelated) SUID binary → root
```

**Why it's bad:** There's no logical connection between the web application and the privilege escalation. The solver could skip the web app entirely if they found the SUID binary through other means.

**What to do instead:** Each step should **require** the output of the previous step. The web app should reveal information or access that makes the privesc vector discoverable or exploitable.

---

## 🚫 Anti-Pattern 8: Unrealistic Service Exposure

**What it looks like:**
```
Port 3306 (MySQL) open to the internet with root/no-password
Port 6379 (Redis) open to the internet with no auth
Port 27017 (MongoDB) open to the internet with no auth
```

**Why it's bad:** While these misconfigurations exist in the wild, exposing database ports directly to the internet is less common than it used to be. Cloud providers and default firewall rules prevent this in most modern deployments.

**What to do instead:** Keep databases on localhost. Let the solver discover them after gaining a shell. The initial attack surface should be limited to web ports (80/443) and SSH (22), unless the story justifies otherwise.

---

## 🚫 Anti-Pattern 9: The Overcomplicated Crypto Challenge

**What it looks like:**
```
Decode this base64 → ROT13 → hex → get a password
```

**Why it's bad:** Multi-layer encoding is not a security vulnerability. It's a puzzle. CTF machines should test offensive security skills, not cryptographic patience.

**What to do instead:** If encryption/encoding is involved, make it realistic — an encrypted config file with a key stored elsewhere, a JWT with a weak secret, a password hash that can be cracked with a wordlist.

---

## 🚫 Anti-Pattern 10: The Impossible-Without-Hints Machine

**What it looks like:**
A machine where the next step is only discoverable if you:
- Know to look in a very specific obscure location
- Use an exact tool with exact non-obvious flags
- Guess a custom port number or URL path

**Why it's bad:** If the solver needs a walkthrough to know *where* to look, the chain isn't logical. The machine should be solvable through systematic enumeration.

**What to do instead:** Every step should be discoverable through standard enumeration:
- `nmap` for external services
- `ss -tlnp` / `netstat` for internal services
- `ps aux` for running processes
- `find` for file permissions
- `ls -la` for directory contents
- `cat` for config files
- `crontab -l` and `/etc/cron.*` for scheduled tasks
- `systemctl list-timers` for systemd timers
