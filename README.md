# CTF Machine Scenario Builder — AI Agent Guide

This repository is a structured template that guides an AI agent to create **realistic, logically connected CTF (Capture The Flag) machine scenarios**. Each scenario describes a vulnerable machine that a penetration tester must compromise step by step.

## How to Use This Repository

When prompting an AI agent, specify the **difficulty level** and optionally a **theme** or **technology stack**. The agent will use the templates and guidelines in this repository to produce a complete CTF machine scenario document.

### Quick Start

```
Build me a [easy|medium|insane] CTF machine scenario.
```

Or with more detail:

```
Build me a medium CTF machine scenario themed around a DevOps startup using Jenkins and Docker.
```

The agent should follow the structure in [`templates/`](templates/) and the quality rules in [`guidelines/`](guidelines/) to generate the scenario.

---

## Repository Structure

```
├── README.md                          # This file — overview and usage
├── guidelines/
│   ├── quality-rules.md               # What makes a good CTF scenario
│   ├── anti-patterns.md               # Common mistakes to avoid
│   └── vulnerability-selection.md     # How to pick realistic vulns per difficulty
└── templates/
    ├── scenario-template.md           # Universal scenario document structure
    ├── easy.md                        # Easy-level constraints and expectations
    ├── medium.md                      # Medium-level constraints and expectations
    └── insane.md                      # Insane-level constraints and expectations
```

---

## Difficulty Levels

| Level | Attack Chain Steps | Vuln Complexity | Rabbit Holes | Expected Solve Time |
|-------|-------------------|-----------------|--------------|---------------------|
| **Easy** | 2–3 | Known CVEs, simple misconfigs | 0–1 | 1–3 hours |
| **Medium** | 3–4 | Chained vulns, service interaction | 1–2 | 3–8 hours |
| **Insane** | 4–6 | Custom exploits, multi-pivot, crypto | 2–3 | 8–24+ hours |

---

## Core Principles

1. **Logical coherence** — Every vulnerability must have a believable reason to exist in the story.
2. **No contrived hints** — No `notes.txt` with passwords, no `credentials.bak`, no backup files or developer notes with credentials.
3. **No lazy privesc** — No `sudo -l` one-liners unless there's a strong narrative reason.
4. **Real-world patterns** — Misconfigurations should mirror what real developers/sysadmins actually do.
5. **Connected chain** — Each step must logically lead to the next through realistic enumeration.
6. **Flags** — Always placed at `/home/<user>/user.txt` and `/root/root.txt`. Flag content must follow the format `MS{<random_hash>}` (e.g., `MS{a1b2c3d4e5f6...}`).
7. **Realistic vulnerabilities** — Labs should use the latest and most common vulnerabilities (e.g., OWASP Top 10) to be as realistic as possible. CVEs must be recent: only `CVE-2025-*` or `CVE-2026-*` are permitted.
8. **Brute-forceable passwords** — When password cracking is part of the chain, passwords must come from well-known wordlists (rockyou.txt, xato-net-10-million-passwords, etc.), never from backup files, developer notes, or planted credential files.

---

## Default Operating System

The available Ubuntu server versions are: **24.04**, **25.04**, and **26.04**.

**Ubuntu 26.04** is the recommended default for all scenarios. Use an older version (24.04 or 25.04) **only** when the scenario specifically requires it — for example, kernel-level CVEs that target older kernel versions.

See [`guidelines/quality-rules.md`](guidelines/quality-rules.md) for the full set of rules.