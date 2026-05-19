# Active Directory & Enterprise Scenarios

AD scenarios are among the highest-value training assets. They simulate real corporate networks and teach techniques used in actual penetration tests.

---

## AD Environment Requirements

Every AD lab must meet these minimum standards:

| Requirement | Detail |
|-------------|--------|
| **Domain naming** | Realistic names: `techcorp.local`, `centralbank.uz`, `retailnet.local`. Generic names (`contoso.local`, `corp.local`) are **prohibited** |
| **User population** | Minimum 30–50 fake users in realistic `first.last` format, organized by department |
| **Password policy** | MinLength 8, Complexity ON — but some service accounts have weak passwords |
| **Service accounts** | SQL service, backup service, web service — SPN registered (Kerberoastable) |
| **Tiered admin model** | Partially implemented Tier 0/1/2 — with deliberate misconfigurations |
| **Group Policy Objects** | Realistic GPOs (mapped drives, logon scripts) |
| **OU structure** | Real departments: IT, Finance, HR, Sales, Engineering, C-Suite |
| **Forest/external trusts** | Required for Hard+ difficulty |

---

## AD Attack Chains by Difficulty

### Easy AD

| Chain | Description |
|-------|-------------|
| AS-REP Roasting → crack → user → credential in share → admin | Pre-auth disabled account → crack → find creds in accessible share |
| Anonymous LDAP bind → description passwords → spray → local admin | Enumerate users via LDAP → find passwords in description fields → password spray |

### Medium AD

| Chain | Description |
|-------|-------------|
| Kerberoasting → crack → lateral → constrained delegation → DC | Crack service account password → move laterally → abuse delegation → domain admin |
| LLMNR/NBT-NS poisoning → NTLMv2 relay → local admin → secretsdump → DA | Network poisoning → relay captured hash → dump secrets → domain compromise |
| Web app → SQLi → DB creds → password reuse → BloodHound → ACL abuse | Web foothold → database credentials → reuse password → discover ACL path |

### Hard AD

| Chain | Description |
|-------|-------------|
| ADCS ESC1/ESC4 → certificate abuse → domain admin | Misconfigured certificate template → forge certificate → DA |
| Unconstrained delegation + PrinterBug → TGT capture → DCSync | Force authentication → capture TGT → replicate directory |
| Shadow Credentials + RBCD → lateral → LAPS password read → DA | Key credential manipulation → resource-based delegation → read LAPS → DA |
| Cross-forest trust abuse → SID history injection → Enterprise Admin | Exploit forest trust → inject SID history → full enterprise compromise |

### Insane AD

| Chain | Description |
|-------|-------------|
| Full corporate simulation | Web DMZ → internal pivot → AD enum → multi-forest compromise |
| gMSA abuse + DPAPI + certificate theft → golden cert → persistent access | Group-managed service account → DPAPI secrets → certificate-based persistence |
| Custom C2 + AV/EDR bypass + real OPSEC | Full red team engagement simulation requiring custom tooling |

---

## AD Design Rules

1. Every AD machine must teach at least one real AD attack technique
2. BloodHound should be useful for discovery but not the only enumeration method
3. Service accounts must have realistic SPNs and Kerberoastable configurations
4. Password policy must be realistic — not every account has a weak password
5. The attack chain must cross at least one trust boundary
6. Hard+ machines should include defensive measures that must be bypassed
