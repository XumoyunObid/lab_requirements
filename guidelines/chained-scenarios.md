# Chained Scenarios — Lateral Movement

Multi-machine scenarios provide real corporate penetration test experience. Students learn pivoting, tunneling, and credential reuse across network segments.

---

## Approved Lateral Movement Techniques

| Technique | Description |
|-----------|-------------|
| **Pass-the-Hash / Pass-the-Ticket** | NTLM hash or Kerberos ticket to access another machine (Impacket, Rubeus) |
| **Credential Reuse** | Password or hash found on one machine works on another |
| **SSH Key Reuse** | Private key found on one server grants access to another |
| **Pivoting/Tunneling** | Chisel, Ligolo-ng, SSH dynamic forwarding — access internal networks |
| **WMI/PSExec/Evil-WinRM** | Windows lateral movement with admin credentials |
| **RDP with Stolen Credentials** | Pass-the-hash or cleartext credentials for RDP access |
| **MSSQL `xp_cmdshell` → Linked Server** | SQL server lateral movement via linked server queries |

---

## Chained Scenario Structure

### Minimum Chain (3 Machines)

| Machine | Role | Description |
|---------|------|-------------|
| **DMZ Box (Web)** | External foothold | Student enters here — public-facing web application |
| **Internal Box** | Pivot target | Reached via tunnel — credential hunting and service exploitation |
| **Target (DC / Crown Jewel)** | Final objective | Domain controller or high-value asset — final exploitation |

### Design Rules

1. Every machine in the chain must contain **clear information** leading to the next machine
2. No machine should be an "island" — every machine is part of the chain
3. Tunneling is **mandatory** — students must learn Chisel, Ligolo-ng, or SSH tunneling
4. Each machine should have its own attack surface, not just be a stepping stone
5. Network segmentation must be realistic — not all machines can reach all others

---

## Network Layout

```
Attacker (VPN: 10.8.0.0/24)
    │
    ▼
DMZ (10.100.X.10) ── Web server, external-facing
    │
    │ [tunnel/pivot]
    ▼
Internal (10.100.X.11) ── Internal services, databases
    │
    │ [credential reuse / lateral movement]
    ▼
Target (10.100.X.20) ── DC / crown jewel server
```

---

## Chained Scenario by Difficulty

### Medium (2 Machines)
- Web server + internal service on separate machines
- Simple credential reuse or SSH key pivot
- One tunnel required

### Hard (3 Machines)
- DMZ → Internal → Target chain
- Multiple lateral movement techniques
- Network segmentation enforced

### Insane (4+ Machines)
- Full enterprise simulation
- Multiple network segments
- Active Directory forest with multiple domains
- Custom C2 and OPSEC considerations
