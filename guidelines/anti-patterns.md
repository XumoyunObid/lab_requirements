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

**What to do instead:** Keep credential discovery realistic: use identity/token abuse, hash cracking from legitimate stores, or runtime/service-level extraction that follows from prior access. When password cracking is needed, use passwords from common brute-force wordlists like `rockyou.txt` or `xato-net-10-million-passwords`.

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

**What to do instead:** If credentials are part of the chain, tie them to realistic system behavior — weak auth boundaries, exposed internal services, reusable service tokens, or crackable hashes from real data stores. Avoid plaintext secrets in `.env`, backups, or convenience files. When brute forcing is required, use passwords that exist in well-known wordlists (rockyou.txt, xato-net-10-million-passwords, SecLists).

---

## 🚫 Anti-Pattern 11: Vulnerability Class Repetition

**What it looks like:**
```
Step 1: SQLi for initial access
Step 2: Another SQLi in internal service
Step 3: SQLi again for privesc metadata
```

**Why it's bad:** Realistic environments can have multiple bugs, but intended chains built on the same class repeatedly feel forced and predictable.

**What to do instead:** Vary intended-path techniques across attack surfaces (web, identity, service trust, automation, local privilege). If the same class appears again, justify it with architecture and make the discovery path explicit.

---

## 🚫 Anti-Pattern 12: Steganography

**What it looks like:**
```
Hidden message inside a JPEG/PNG image on the web server
Use steghide/stegsolve to extract a password
```

**Why it's bad:** Steganography is not a penetration testing technique. No real corporate pentest involves extracting data from images. This is a puzzle, not a security vulnerability.

**What to do instead:** Use realistic data discovery — config files, database queries, API responses, service enumeration.

---

## 🚫 Anti-Pattern 13: Brute-Force as the Only Path

**What it looks like:**
```
Login page with no rate limiting
Only way in is to brute-force the password
No other enumeration leads anywhere
```

**Why it's bad:** If the entire foothold depends on brute-forcing without any other enumeration path, the machine tests patience, not skill. Real pentest findings come from misconfigurations and vulnerabilities, not pure brute-force.

**What to do instead:** Brute-force can be part of a chain (e.g., credential stuffing with a leaked list), but must never be the sole discovery method. Provide realistic enumeration paths.

---

## 🚫 Anti-Pattern 14: PATH Hijacking (Simple)

**What it looks like:**
```
export PATH=/tmp:$PATH
Create fake binary in /tmp → run vulnerable script → root
```

**Why it's bad:** Simple PATH hijacking where you prepend `/tmp` to PATH and replace a binary is almost never seen in real corporate environments. Modern scripts use absolute paths, and this is a CTF-only pattern.

**What to do instead:** Use dependency hijacking in custom applications (e.g., Python/Node module path injection, DLL hijacking in writable directories) which reflects real-world attack vectors.

---

## 🚫 Anti-Pattern 15: LD_PRELOAD as Sole Privesc

**What it looks like:**
```
sudo -l shows: env_keep += LD_PRELOAD
Compile shared library → LD_PRELOAD=/tmp/evil.so sudo <command> → root
```

**Why it's bad:** `env_keep += LD_PRELOAD` is not found in real sudoers configurations. This is a teaching example that became a CTF cliché. No sysadmin deliberately keeps LD_PRELOAD in the sudo environment.

**What to do instead:** Use real shared library injection paths — writable library directories in the search path, writable RPATH in custom binaries, or legitimate LD_PRELOAD configurations in application startup scripts.

---

## 🚫 Anti-Pattern 16: SUID on Standard Utilities

**What it looks like:**
```
find / -perm -4000 2>/dev/null
/usr/bin/find (SUID)
/usr/bin/vim (SUID)
/usr/bin/less (SUID)
/usr/bin/nmap (SUID)
```

**Why it's bad:** No system administrator sets the SUID bit on `find`, `vim`, `less`, or `nmap`. These are GTFOBins entries that exist only in CTFs. This teaches students to rely on an unrealistic pattern they will never encounter on a real engagement.

**What to do instead:** If SUID binaries are part of the chain, use custom binaries that require reverse engineering (Medium+) or legitimate SUID programs with known vulnerabilities.

---

## 🚫 Anti-Pattern 17: Trivia and Puzzle Challenges

**What it looks like:**
```
Solve a math puzzle to get the next password
Decode ROT13 → base64 → hex → get credentials
Answer a riddle on the web page to unlock admin
```

**Why it's bad:** Penetration testing is not a puzzle game. Multi-layer encoding, math challenges, sudoku, and riddles have no place in a realistic attack simulation. These test general knowledge, not security skills.

**What to do instead:** Use real cryptographic weaknesses (weak JWT secrets, predictable tokens, padding oracle) or realistic encoding scenarios (base64-encoded API tokens that need to be decoded once as part of normal API interaction).

---

## 🚫 Anti-Pattern 18: Social Engineering Hints

**What it looks like:**
```
Hidden HTML comment: "<!-- admin password: Summer2024! -->"
Web page with subtle hint: "Did you check the admin panel?"
robots.txt pointing to /secret-admin as the ONLY discovery path
```

**Why it's bad:** Planting obvious hints in HTML comments or using social engineering cues is not realistic enumeration. While `robots.txt` is a valid recon technique, it should not be the sole path to discovery.

**What to do instead:** Let the solver discover paths through standard web fuzzing (gobuster, ffuf), API enumeration, or service fingerprinting. Information should come from realistic sources — error messages, API responses, version disclosures.

---

## 🚫 Anti-Pattern 19: Prohibited Content

The following content is **strictly prohibited** in all machines:

- **Real malware** — even in disabled state, never include actual malware samples
- **Cracked/pirated software** — never use unlicensed commercial software
- **Backdoors** — no hidden access methods for the machine engineer
- **Ransomware simulation** — no encryption of user files
- **Botnet/C2 client code** — no command-and-control infrastructure
- **Undocumented internet dependency** — if a machine requires internet access, this must be explicitly documented in the technical requirements

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
