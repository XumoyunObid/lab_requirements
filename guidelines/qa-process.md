# QA and Submission Process

Every machine goes through a multi-stage review process before release. No machine is published without passing all stages.

---

## QA Stages

### 1. Self-Review

The author solves their own machine following the walkthrough. Resource usage and reset functionality are verified.

### 2. Peer Review

A different engineer solves the machine with **zero hints**. Solve time is measured and compared against the target for the difficulty level.

### 3. Realism Check

The lead engineer asks: **"Would this happen in the real world?"** — yes/no for each chain step. Any "no" answer requires redesign.

### 4. Technical Review

Check for:
- Unintended paths (shortcuts that bypass the intended chain)
- Stability issues (services crashing, race conditions)
- Edge cases (what happens if the student does X before Y?)

### 5. Beta Release

50–100 students test the machine for 1–2 weeks. Feedback is collected on:
- Difficulty accuracy
- Clarity of enumeration paths
- Stability under load
- Unintended solutions

### 6. Public Release

Machine is opened to all students with monitoring for:
- Solve rates by difficulty level
- Common failure points
- Performance issues

---

## Rejection Criteria

A machine is **immediately rejected** if any of the following apply:

| Criterion | Description |
|-----------|-------------|
| ⛔ Blacklisted privesc | Uses any banned technique (sudo -l GTFOBins, writable `/etc/passwd`, SUID standard utilities, etc.) |
| ⛔ Brute-force/guessing foothold | Foothold requires brute-force or random guessing as the sole path |
| ⛔ Too easy for stated difficulty | Machine is solvable in under 15 minutes — difficulty label is wrong |
| ⛔ Missing/low-quality walkthrough | No walkthrough, or walkthrough missing required sections |
| ⛔ Unrealistic scenario | Attack chain would not occur in a real corporate environment |
| ⛔ Plagiarism | Direct copy from another platform (HackTheBox, TryHackMe, etc.) |
| ⛔ Resource limit exceeded | Machine exceeds resource limits without lead engineer approval |
| ⛔ Unintended easy path | Unintended shortcut exists and is not blocked |
