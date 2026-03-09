---
layout: default
title: Getting Started
nav_order: 1
description: "Prerequisites, installation, and your first investigation with Win-Investigator"
---

# Getting Started
{: .no_toc }

Get Win-Investigator running and diagnose your first server in minutes.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

### 1. PowerShell on Your Local Machine

Windows PowerShell 5.1+ or PowerShell 7+ is required on the machine where you run Copilot CLI.

### 2. PowerShell Remoting on Target Servers

{: .important }
> This is a **one-time setup** per server. You need local admin or Domain Admin rights on the target.

```powershell
# Run this on each target Windows Server
Enable-PSRemoting -Force
winrm quickconfig -q
```

Verify it's enabled:

```powershell
Test-WSMan server01 -UseSSL   # Replace with your server name
```

### 3. Network Access

Your machine must be able to reach the target server on:

| Port | Protocol | Purpose |
|------|----------|---------|
| 5986 | HTTPS | WinRM (encrypted) |

Ensure firewalls and routing allow this traffic.

### 4. Admin Rights on Target Servers

Your user account (or the credentials you provide) must have **local administrator** or equivalent rights on each target server.

### 5. Copilot CLI Installed

```bash
# Install Copilot CLI (if not already installed)
npm install -g @github/copilot-cli

# Authenticate
copilot auth login
```

---

## Your First Investigation

Once prerequisites are in place, open your Copilot CLI and ask a question:

```bash
copilot "What is going on with server01?"
```

Win-Investigator will:

1. Parse your question to identify **server01** and determine a **general health check**
2. Connect to server01 using your current Windows credentials
3. Run overview diagnostics (OS info, uptime, disk, memory, services)
4. Return a structured report

### Example Output

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 WIN-INVESTIGATOR REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SERVER: server01
STATUS: 🟡 Warning
TIMESTAMP: 2026-03-09T14:30:00Z

───────────────────────────────────────────────────
FINDINGS
───────────────────────────────────────────────────

🔴 Disk Space Critical
  C: drive is 92% full (4.6 GB free of 60 GB total)
  Impact: Apps may fail to write temp files, event logs may not record
  Action: Delete temp files, archive old logs, or expand volume

🟡 Memory Usage Elevated
  Currently 78% (12.5 GB of 16 GB), SQL Server using 8.2 GB
  Impact: System may slow down under load; disk paging increases
  Action: Monitor trends; if persistent, check SQL for memory leaks

🟢 Critical Services Running
  SQL Server Agent: Running
  IIS World Wide Web Publishing: Running
  Windows Defender: Running

───────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────

server01 is mostly healthy but has two concerns: disk space is critically
low at 92% full, and memory is elevated at 78%. Prioritize freeing disk
space to prevent application failures, then monitor memory trends.

Next steps: Clean C:\Temp and C:\Windows\Temp directories, then review
IIS logs for archival.
```

---

## Next Steps

Now that you've run your first investigation:

- [Learn question patterns and credential handling →](/win-investigator/usage)
- [See all available diagnostics →](/win-investigator/diagnostics)
- [Browse example sessions →](/win-investigator/examples)

---

_Built by the Win-Investigator team._
