---
layout: default
title: Troubleshooting
nav_order: 5
description: "Common errors and fixes when using Win-Investigator"
---

# Troubleshooting
{: .no_toc }

Solutions for common issues you may encounter.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Server Unreachable

```
❌ Error: Unable to connect to server01
```

### Checklist

1. **Hostname/IP is correct** — Verify the name resolves: `nslookup server01`
2. **Server is online** — `ping server01`
3. **PowerShell remoting is enabled** on the target:
   ```powershell
   # Run on the target server
   Enable-PSRemoting -Force
   winrm quickconfig -q
   ```
4. **Firewall allows WinRM HTTPS** — Port 5986 must be open
5. **Network connectivity** exists between your machine and the server

{: .note }
> If the server responds to `ping` but not `Test-WSMan`, the issue is WinRM configuration, not network connectivity.

---

## Access Denied

```
❌ Error: Access denied on server01
```

### Checklist

1. **Your user is an admin** on the target server
2. **Credentials are correct** (if using explicit mode)
3. **User is in the Administrators group** on the target
4. **Account is not disabled or locked out**

### Fix: Use Explicit Credentials

```bash
copilot "Check server01 with domain\admin credentials"
```

### Fix: Verify Group Membership

```powershell
# Run on the target server to check if a user is admin
net localgroup Administrators
```

---

## WinRM Not Responding

```
❌ Error: WinRM is not responding
```

### Fix: On the Target Server

```powershell
# Check if WinRM service is running
Get-Service WinRM

# Start it if needed
Start-Service WinRM

# Re-enable PowerShell remoting
Enable-PSRemoting -Force
```

### Fix: Check Firewall Rules

```powershell
# On the target server — check for WinRM HTTPS firewall rule
Get-NetFirewallRule -DisplayName "WinRM HTTPS" -ErrorAction SilentlyContinue | Select-Object DisplayName, Enabled
# Also check for built-in HTTPS rule
Get-NetFirewallRule -Name "WINRM-HTTPS-In-TCP" -ErrorAction SilentlyContinue | Select-Object DisplayName, Enabled
```

### Fix: Certificate Handling (IP Addresses and Self-Signed Certs)

When connecting to IP addresses or servers with self-signed certificates, use session options:

```powershell
# On YOUR machine — use SkipCACheck and SkipCNCheck (no TrustedHosts modification needed)
$SessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
New-PSSession -ComputerName "server01" -UseSSL -Port 5986 -SessionOption $SessionOption
```

{: .note }
> `-SkipCACheck` and `-SkipCNCheck` handle self-signed certificates and IP address connections without requiring TrustedHosts modification.

---

## Command Timed Out

If diagnostics hang or timeout:

1. **Target server is under heavy load** — CPU or memory may be maxed out
2. **Network latency is high** — Check for routing issues or congestion
3. **Try a simpler diagnostic first** — A general health check is lighter than a deep disk analysis
4. **Check server responsiveness** — Can you run basic commands like `ping` or `Test-WSMan`?

{: .note }
> Some diagnostics (like finding large files across an entire volume) can take several minutes on large servers. This is expected.

---

## DNS Resolution Failure

```
❌ Error: server01 cannot be resolved
```

### Fix

1. Try the **IP address** instead of hostname
2. Check your **DNS settings**: `nslookup server01`
3. Verify DNS is working: `nslookup google.com`
4. If using a short name, try the **FQDN**: `server01.contoso.com`

---

## WinRM HTTPS Certificate Issues

```
❌ Error: Server certificate invalid
```

This occurs when using HTTPS (port 5986) and the certificate is self-signed or expired.

### Fix

For testing, you can skip certificate verification:

```powershell
$sessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
Enter-PSSession -ComputerName server01 -UseSSL -SessionOption $sessionOption
```

{: .warning }
> Skipping certificate checks reduces security. Only use this for testing in trusted environments. In production, configure proper certificates.

---

## Partial Results

If Win-Investigator returns partial data or some checks fail:

- **Some cmdlets may not be available** on older Windows Server versions
- **Permissions may be insufficient** for certain data (e.g., Security event log)
- **Modules may not be installed** (e.g., ServerManager on non-Server SKUs)

Win-Investigator handles these gracefully by falling back to alternative methods (e.g., WMI instead of Storage cmdlets) and reporting what it could collect.

---

## Escalation Guide

When Win-Investigator identifies issues but you need human action:

| Issue | Escalate To | What to Tell Them |
|-------|-------------|-------------------|
| Service needs restart | On-call admin | Service name, current state, crash history |
| Disk space critical | Infrastructure | Drive letter, free space, cleanup candidates |
| Database issues | DBA | SQL instance, error events, resource usage |
| Security events | Security team | Event IDs, timestamps, affected accounts |
| Hardware failure | Infrastructure | SMART data, disk health status, error counts |
| Application crash | App owner | Process name, PID, crash events, memory usage |
| Network issues | Network admin | Adapter status, DNS config, port test results |

{: .important }
> Always include the Win-Investigator report when escalating. It provides the specific data and timestamps that responders need.

---

_Built by the Win-Investigator team._
