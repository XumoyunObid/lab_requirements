# Walkthrough Standard

Every machine must include a complete walkthrough. Walkthroughs are internal documents — they are **not** shown to students.

---

## Required Structure

Every walkthrough must contain the following sections in order:

### 1. Metadata

| Field | Description |
|-------|-------------|
| Machine name | Official machine name |
| Difficulty | Easy / Medium / Hard / Insane |
| Operating System | OS and version |
| Author | Engineer name |
| Date | Creation/submission date |
| Skills covered | List of skills the student learns |

### 2. Attack Chain Diagram

Visual representation of the full attack chain using ASCII art or Mermaid diagram format.

```
Anonymous
    │  nmap scan → identify web service
    │  web fuzzing → find vulnerable endpoint
    ▼
www-data (via SQLi → RCE)
    │  config file → database credentials
    │  credential reuse → SSH access
    ▼
developer → user.txt
    │  systemd timer → writable script
    │  inject reverse shell → root
    ▼
root → root.txt
```

### 3. Step-by-Step Solution

For each step:
- **Screenshot** of the relevant output
- **Exact commands** used
- **Explanation** of what the command does and why it works
- **"What was learned?"** — one sentence summarizing the skill practiced

### 4. Alternative Paths

Document any valid alternative approaches to solving the machine (if they exist).

### 5. Rabbit Holes List

For each rabbit hole:
- What the student will find
- Why it looks promising
- Why it fails
- How long an average student might spend on it

### 6. Tools Used

Complete list of all tools required or useful for solving the machine.

### 7. MITRE ATT&CK Mapping

Map each attack chain step to the corresponding MITRE ATT&CK technique ID (e.g., `T1190` — Exploit Public-Facing Application).

### 8. CVE/CWE References

List all CVEs and CWEs relevant to the machine's vulnerabilities with links to advisories.
