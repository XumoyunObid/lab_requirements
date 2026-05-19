# Technical Requirements

This document defines deployment, resource, networking, flag, and reset standards for all machines.

---

## Deployment

| Requirement | Detail |
|-------------|--------|
| **Platform** | Proxmox VE (LXC containers and QEMU VMs) |
| **Format** | VM template (cloned per user) |
| **Naming** | English, lowercase, hyphen-separated (e.g., `web-vuln-01`, `ad-forest-dc01`) |
| **Machine ID** | Every machine must have a unique `machine_id` |

---

## Resource Limits

Standard limits — exceeding these requires lead engineer approval:

| Difficulty | CPU | RAM | Disk |
|------------|-----|-----|------|
| **Easy** | 1–2 cores | 1–2 GB | 8–15 GB |
| **Medium** | 1–2 cores | 1–2 GB | 8–15 GB |
| **Hard** | 2–4 cores | 2–4 GB | 15–30 GB |
| **Insane / Chained** | 4–8 cores | 4–8 GB | 30–60 GB (entire chain) |
| **AD Domain Controller** | 2 cores minimum | 4 GB minimum | — |

---

## Network

| Component | Detail |
|-----------|--------|
| **User isolation** | Each user gets an isolated VLAN (`vmbr2` OVS bridge) |
| **VLAN → Subnet mapping** | `10.100.{vlan_id}.0/24` |
| **VPN access** | OpenVPN, `10.8.0.0/24` |
| **Chained machine IPs** | `10.100.X.10`, `.11`, `.20` |
| **Internet** | Restricted by default — only enabled when explicitly required (e.g., apt repository) |
| **DNS** | Local resolver (nslookup must work) |

---

## Flag System

### Format

```
MAHADSEC{<32_hex_characters>}
```

Example: `MAHADSEC{a7f3b9c2e1d8f4a6b5c3d2e1f0a9b8c7}`

### Rules

| Rule | Detail |
|------|--------|
| **Uniqueness** | Every flag is unique per user (injected via cloud-init/API at runtime) |
| **Standalone machines** | 2 flags: `user.txt` and `root.txt` |
| **Chained machines** | Per-machine flags or final flag only (designer's choice) |
| **No hardcoding** | Flags must **never** be hardcoded — generated at runtime |
| **Permissions** | Each flag file is readable only by the appropriate user |

### Locations

| Flag | Path | Access |
|------|------|--------|
| `user.txt` | `/home/<user>/user.txt` | Compromised user only (and root) |
| `root.txt` | `/root/root.txt` | Root only |

---

## Auto-Reset

| Requirement | Detail |
|-------------|--------|
| **Clean snapshot** | Take snapshot before deployment — used for reset |
| **Reset time** | Must complete in under 30 seconds |
| **Flag persistence** | Flags survive reset (not regenerated) |
| **Cleanup targets** | `.bash_history`, `/tmp`, logs are cleaned — attack surface preserved |
