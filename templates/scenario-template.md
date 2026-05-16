# CTF Machine: [Name]

## Overview

**Name:** [Machine Name]
**Difficulty:** [Easy | Medium | Hard | Insane]
**OS:** [Operating System and Version — default: Ubuntu 26.04 unless older version justified for kernel CVEs]
**Domain:** `[machine-name.ms]`
**Theme:** [One-line description of the scenario's real-world context]

**Story:** [2-4 sentences describing who built this system, why it exists, and what mistakes were made. The story must explain WHY each vulnerability exists — careless developer, rushed deployment, legacy system, etc.]

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | [Web server] → [Application framework] (port [N]) |
| Database | [Database] ([binding], [auth status]) |
| [Other components] | [Details] |
| Open Ports | [List all externally accessible ports with services] |
| App runs as | `[username]` user |

<!-- Add rows for each significant component: process managers, message queues, caches, containers, etc. -->

---

## Attack Chain

### 1 → [Step Title] — [From Access Level] → [To Access Level]

[2-3 sentences describing what the solver does and discovers at this step.]

- [Enumeration technique 1]
- [Enumeration technique 2]
- [Key discovery or vulnerability identification]

**Exploitation:**
```bash
# [Command or tool usage]
```

**Why it works:** [1-2 sentences explaining why this vulnerability exists in the context of the story.]

---

### 2 → [Step Title] — [From Access Level] → [To Access Level]

[Repeat the pattern above for each step in the chain.]

---

### 3 → [Step Title] — [From Access Level] → [To Access Level]

[Continue until root is achieved.]

---

## Rabbit Holes

<!-- Required for Medium (1-2), Hard (2), and Insane (2-3). Optional for Easy (0-1). -->

### Rabbit Hole 1: [Name]

**What the solver finds:** [Description of the misleading discovery]
**Why it looks promising:** [Why a solver would investigate this]
**Why it's a dead end:** [Technical reason it doesn't work]
**What it teaches:** [Skill or lesson the solver gains from investigating this]

---

## Flags

Flag format: `MS{<random_hash>}` (e.g., `MS{a1b2c3d4e5f67890abcdef1234567890}`)

| Flag | Location | Content | Access |
|------|----------|---------|--------|
| user.txt | /home/[user]/ | `MS{<unique_random_hash>}` | [username] |
| root.txt | /root/ | `MS{<unique_random_hash>}` | root |

---

## Full Chain Diagram

```
Anonymous
    │  [enumeration steps]
    │  [vulnerability exploited]
    ▼
[user]  →  user.txt
    │  [enumeration steps]
    │  [vulnerability exploited]
    ▼
root  →  root.txt
```

---

## Implementation Notes

<!-- These notes are for the lab builder who will set up the actual machine. -->

### Packages to Install
- [List all packages needed]

### Configuration Changes
- [List all non-default configurations]

### Files to Create
- [List all custom files, scripts, configs]

### Services to Enable
- [List all services and their configurations]

### User Accounts
| Username | Password | Shell | Groups | Purpose |
|----------|----------|-------|--------|---------|
| [user] | [password] | /bin/bash | [groups] | [role in the story] |

### Verification Checklist
- [ ] External port scan shows only expected ports
- [ ] Initial exploit works from external attacker perspective
- [ ] Each chain step is achievable from the previous step
- [ ] Rabbit holes are convincing but definitively dead ends
- [ ] Flags are in correct locations with correct permissions
- [ ] No unintended shortcuts exist (test with automated privesc tools)
