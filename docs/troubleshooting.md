---
layout: default
title: Troubleshooting
nav_order: 5
description: "Fixes for common setup and connection issues. Start here if something isn't working."
---

# Troubleshooting
{: .no_toc }

**Something not working?** We've got solutions.
{: .fs-6 .fw-300 }

{: .important }
> **Security Note:** Never type passwords in the Copilot CLI chat. Always create your credential 
> in PowerShell **before** running `gh copilot`: `$credential = Get-Credential`

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## First Time Setup Issues

### `gh: command not found`

**Problem:** PowerShell doesn't recognize the `gh` command.

**Cause:** GitHub CLI is not installed or not in your PATH.

**Solution:**

1. Verify it installed:
   ```bash
   gh --version
   ```

2. If that fails:
   - Download from: https://cli.github.com/
   - Or reinstall via PowerShell:
     ```powershell
     winget install GitHub.cli
     ```

3. **Restart PowerShell** after installing, then try again.

---

### `Error: not authenticated` or `not logged in`

**Problem:** `gh auth status` shows "You are not logged in to any GitHub hosts."

**Cause:** You skipped the authentication step (Step 2 of Getting Started).

**Solution:**

```bash
gh auth logout
gh auth login
```

Follow the prompts to authenticate via web browser.

---

### `copilot extension not found`

**Problem:** `gh extension list | grep copilot` returns nothing.

**Cause:** The Copilot CLI extension is not installed.

**Solution:**

```bash
gh extension install github/gh-copilot
```

Verify it worked:

```bash
gh extension list
# Should show: github/gh-copilot
```

---

### `Error: Extension not available`

**Problem:** `gh extension install github/gh-copilot` fails with: "Extension not available."

**Cause:** Usually a network or authentication issue.

**Solution:**

1. Verify you're authenticated:
   ```bash
   gh auth status
   ```

2. If needed, re-authenticate:
   ```bash
   gh auth logout
   gh auth login
   ```

3. Try the install again:
   ```bash
   gh extension install github/gh-copilot
   ```

---

## Credential Issues

### `$credential` variable not found

**Problem:** The agent says `$credential` is not defined, or asks you to create it.

**Cause:** You need to create the `$credential` variable in your PowerShell session **before** running `gh copilot`.

**Solution:**

Run this in your PowerShell session (not inside Copilot chat):

```powershell
$credential = Get-Credential
```

A secure Windows dialog will open. Enter your username and password there, then start Copilot:

```powershell
gh copilot
```

The agent will automatically detect and use the `$credential` variable for remote connections.

{: .note }
> The `$credential` variable persists for the lifetime of your PowerShell session. You only need to set it once per session.

### Credentials keep being asked for the same server

**Problem:** Every investigation requires you to create `$credential` again.

**Cause:** The `$credential` variable only lasts for the current PowerShell session. If you close and reopen PowerShell, you need to set it again.

**Solution:**

Set `$credential` once at the start of your PowerShell session, then run as many `gh copilot` investigations as you need:

```powershell
# Set once per session (before running gh copilot):
$credential = Get-Credential

# Now run Copilot — credentials are reused automatically:
gh copilot
# ? "Check server01"
# ? "Check server02"
# ? "Check server03"
```

If you open a new PowerShell window, run `$credential = Get-Credential` again before starting Copilot.

### "Wrong username or password"

**Problem:** You created `$credential` but connections fail with "Access denied".

**Solution:**

1. **Verify the username format** when creating `$credential`:
   - Domain account: `domain\username` or `username@domain.com`
   - Local account: `.\username` or `ServerName\username`
   - Azure VM local: `VMName\AdminUser`
   - Azure VM with Azure AD: `user@domain.com`

   ```powershell
   # Re-create with the correct username format:
   $credential = Get-Credential
   ```

2. **Check Caps Lock** is off when typing password in the credential dialog

3. **Re-create the credential** — Run `$credential = Get-Credential` again and be careful with the password

4. **Verify the account is not locked**:
   ```powershell
   # On domain controller:
   Get-ADUser username -Properties LockedOut
   ```

5. **Check the account has admin rights** on the target server

---

## Connection Issues

### `Test-WSMan: Unable to connect to the remote host`

**Problem:** `Test-WSMan server01 -UseSSL` fails.

**Cause:** PowerShell remoting is not enabled on the target server, or firewall blocks port 5986.

**Solution (on target server):**

1. **Re-enable PowerShell remoting:**

   ```powershell
   Enable-PSRemoting -Force
   winrm quickconfig -q
   ```

2. **Start the WinRM service:**

   ```powershell
   Start-Service WinRM
   Set-Service WinRM -StartupType Automatic
   ```

3. **Open Windows Firewall for WinRM HTTPS:**

   ```powershell
   New-NetFirewallRule -DisplayName "WinRM HTTPS" `
     -Name "WINRM-HTTPS-In-TCP" -Profile Any -LocalPort 5986 `
     -Protocol TCP -Action Allow -Direction Inbound -ErrorAction SilentlyContinue
   ```

4. **From your machine, test again:**

   ```powershell
   Test-WSMan server01 -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck
   ```

   **Expected output:**
   ```
   wsmid           : http://schemas.dmtf.org/wbem/wsman/identity/identity.xsd
   ProtocolVersion : http://schemas.dmtf.org/wbem/wsmanidentity/1.0.0
   ```

---

### `Error: Cannot resolve server hostname`

**Problem:** `Test-WSMan server01` fails with: "Cannot find the target computer name."

**Cause:** DNS can't resolve the server name, or the server is offline.

**Solution:**

1. **Verify the server is online:**
   ```powershell
   ping server01
   ```

2. **Check DNS can resolve it:**
   ```powershell
   nslookup server01
   ```

3. **Try the FQDN (fully qualified domain name):**
   ```powershell
   Test-WSMan server01.contoso.com -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck
   ```

4. **Try the IP address:**
   ```powershell
   Test-WSMan 192.168.1.10 -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck
   ```

5. **Check your DNS settings:**
   ```powershell
   ipconfig /all | grep DNS
   ```

---

### `Error: Access denied`

**Problem:** Win-Investigator can reach the server but gets "access denied" errors.

**Cause:** Your user account doesn't have admin rights on the target.

**Solution 1: Verify admin rights**

On **target server**, check if your user is admin:

```powershell
net localgroup Administrators
```

Your user should be listed. If not, and you have local admin, add them:

```powershell
net localgroup Administrators domain\username /add
```

**Solution 2: Use explicit credentials**

Create a `$credential` variable before starting Copilot:

```powershell
$credential = Get-Credential
gh copilot
```

```
? "Check server01"
```

The agent will automatically use the `$credential` for the connection.

---

### `Error: Server certificate invalid`

**Problem:** Connection fails with: "Server certificate validation failed."

**Cause:** WinRM is using a self-signed certificate (normal) but your machine doesn't trust it.

**Solution:**

All Win-Investigator connections use `-SkipCACheck -SkipCNCheck`, which handles this automatically. If you see this error anyway:

```powershell
# From your machine, test with the right flags:
$SessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
Test-WSMan server01 -UseSSL -Port 5986 -SessionOption $SessionOption
```

---

## After Connection Issues

### `Error: Unable to connect to server01` (from Win-Investigator)

**Checklist:**

{: .important }
> Verify each step in order before moving to the next:

1. ✅ Server name/IP is correct: `ping server01`
2. ✅ Server is online and reachable
3. ✅ PowerShell remoting enabled on target: `Enable-PSRemoting -Force`
4. ✅ WinRM service running on target: `Get-Service WinRM`
5. ✅ Firewall allows port 5986 on target
6. ✅ Test from your machine: `Test-WSMan server01 -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck`
7. ✅ You have admin rights on target (or use explicit credentials)

If all of these pass, try your investigation again.

---

### `Error: WinRM is not responding`

**Problem:** Win-Investigator times out or fails with WinRM errors.

**Solution:**

1. **On target server, restart WinRM:**

   ```powershell
   Stop-Service WinRM -Force
   Start-Service WinRM
   ```

2. **Re-enable PowerShell remoting:**

   ```powershell
   Enable-PSRemoting -Force
   winrm quickconfig -q
   ```

3. **From your machine, verify Test-WSMan works:**

   ```powershell
   Test-WSMan server01 -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck
   ```

4. **Then try your investigation again.**

---

## Runtime Issues

### `Error: Command timed out`

**Problem:** Win-Investigator hangs or returns "timed out."

**Cause:** Target server is under heavy load, or network latency is high.

**Solution:**

1. Check if the target server is responsive:
   ```powershell
   ping server01
   ```

2. Check server load (from target):
   ```powershell
   Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
   ```

3. Try a simpler diagnostic first:
   ```
   ? "Check server01"  ← Lighter than specific diagnostics
   ```

4. If diagnostics consistently time out, the server may be overloaded. Escalate to on-call admin.

{: .note }
> Large disk scans or event log searches can take 2-5 minutes on large servers. This is expected.

---

### `Error: Partial results`

**Problem:** Win-Investigator returns incomplete data.

**Cause:** Some checks failed due to:
- Insufficient permissions (can't read Security event log, for example)
- PowerShell modules not available on target
- Older Windows Server versions with fewer cmdlets

**Solution:**

Win-Investigator handles this gracefully. It reports what it could collect and notes what failed. This is normal and doesn't prevent useful diagnostics.

---

## Network / Firewall Issues

### `Firewall blocks port 5986`

**On target server, verify the rule exists:**

```powershell
Get-NetFirewallRule -DisplayName "*WinRM*" | Select-Object DisplayName, Enabled
```

If you don't see an enabled rule for port 5986, create one:

```powershell
New-NetFirewallRule -DisplayName "WinRM HTTPS" `
  -Name "WINRM-HTTPS-In-TCP" -Profile Any -LocalPort 5986 `
  -Protocol TCP -Action Allow -Direction Inbound
```

Then test from your machine:

```powershell
Test-WSMan server01 -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck
```

---

### Servers on Different Networks

If your machine and target servers are on different networks (e.g., datacenter behind a proxy):

1. Ensure network routing permits HTTPS traffic on port 5986
2. Verify no proxy is blocking the connection (proxies often block remoting)
3. Try the IP address instead of hostname
4. Contact network admin if firewalls block the connection

---

## Escalation Guide

When Win-Investigator identifies issues but can't fix them, here's who to contact:

| Issue | Escalate To | What to Share |
|-------|-------------|-------------------|
| Service needs restart | On-call admin | Server name, service name, error events |
| Disk critically full | Infrastructure | Drive letter, free space, largest folders |
| Database issues | DBA | SQL instance, memory usage, error events |
| Security events | Security team | Event IDs, timestamps, affected accounts |
| Hardware failure | Infrastructure | Disk health, SMART data, error counts |
| Application crash | App owner | Process name, PID, error events |
| Network issues | Network admin | Adapter status, DNS config, open ports |

Always share the Win-Investigator report — it gives the specific data responders need.

---

_Built by the Win-Investigator team._
