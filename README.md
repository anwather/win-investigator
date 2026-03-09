# Win-Investigator

**AI-driven Windows Server troubleshooting via the Copilot CLI.**

Ask natural language questions about your Windows Servers and get clear, actionable diagnostic reports. No complex commands to memorize — just ask.

---

## What It Does

Win-Investigator diagnoses Windows Server issues by:

1. **Listening** to your question: "What is going on with server01?"
2. **Connecting** to the target server via PowerShell remoting
3. **Running focused diagnostics** based on your concern (disk space, memory, services, network, etc.)
4. **Reporting findings** in a clear, prioritized format with suggested next steps

It answers questions like:

- "What is going on with server01?"
- "Why is server01 running out of disk space?"
- "Is the SQL service running on server02?"
- "server03 is slow — what's using the resources?"
- "Can you reach server04?"

---

## Quick Start

### Prerequisites

1. **Windows PowerShell 5.1+ or PowerShell 7+** on your local machine
2. **PowerShell remoting enabled** on target servers:
   ```powershell
   # On target server, run once:
   Enable-PSRemoting -Force
   winrm quickconfig -q
   ```
3. **Network access** to target servers (port 5986 for HTTPS)
4. **Admin rights** on target servers (or sufficient permissions for diagnostics)
5. **Copilot CLI** installed and authenticated

### Using Win-Investigator

In your Copilot CLI session:

```bash
# Ask about a server
copilot "What is going on with server01?"

# Ask about a specific concern
copilot "server01 is running out of disk space. What can I delete?"

# Ask about a specific diagnostic
copilot "Is the SQL service running on server02?"

# Use explicit credentials
copilot "Check server03 memory with domain\admin credentials"
```

### What You Get

A structured report like this:

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
  C: is 92% full (4.6 GB free of 60 GB)
  Impact: May cause app failures, temp file errors
  Action: Clean temp folders, archive logs, or expand volume

🟡 Memory Usage High
  78% of 16 GB in use (SQL Server using 8.2 GB)
  Impact: Performance may degrade under load
  Action: Monitor trends; check for memory leaks

───────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────

server01 is mostly healthy but disk is critically low and memory is elevated. 
Prioritize freeing disk space, then monitor memory trends.

Next steps: Clean C:\Temp and C:\Windows\Temp, then review IIS logs for archival.
```

---

## Credentials

### Default (Current User)

Win-Investigator uses your current Windows user identity by default. No passwords to enter — just works if you have network and admin access to the target server.

```bash
copilot "Check server01"
# Uses: your-domain\your-user (implicit)
```

### Explicit Credentials

If you need to use different credentials:

```bash
copilot "Check server01 with domain\admin credentials"
# Prompts: Enter password for domain\admin
# Uses: domain\admin for this check
```

---

## Available Diagnostics

Win-Investigator can run these focused checks based on your question:

| Concern | Command Example | What You Get |
|---------|-----------------|-------------|
| **General Health** | "What is going on with server01?" | OS info, uptime, disk, memory, services snapshot |
| **Disk Space** | "server01 is out of disk space" | Drive usage, free space, largest folders, cleanup suggestions |
| **Memory/CPU** | "server01 is slow" | Memory %, CPU %, top processes by resource usage |
| **Services** | "Is SQL running?" | Service status, startup type, recent errors, event log entries |
| **Network** | "Can you reach server01?" | Network adapters, IP config, connectivity tests, open ports |
| **Performance Baseline** | "Get baseline performance on server01" | Memory, CPU, disk, process snapshot for trend comparison |

---

## Prerequisites Detail

### 1. PowerShell Remoting on Target Servers

**One-time setup per server** (requires local admin or Domain Admin):

```powershell
# Run this on each target Windows Server
Enable-PSRemoting -Force
winrm quickconfig -q
```

**Verify it's enabled:**
```powershell
Test-WSMan server01 -UseSSL  # Replace with actual server name
```

If this fails:
- Check the server is online and reachable
- Verify WinRM service is running: `Get-Service WinRM`
- Check Windows Firewall allows port 5986 (HTTPS)

### 2. Network Access

- Target server must be reachable from your machine (ping, DNS, or IP)
- Firewall must allow traffic on port 5986 (HTTPS)
- If servers are on different networks, ensure routing and firewall rules permit the connection

### 3. Admin Access

- Your user must have local admin or equivalent rights on the target server
- Or, you must be able to provide credentials for a user who does (use explicit credentials mode)

### 4. Copilot CLI Installed

Win-Investigator runs within the Copilot CLI. You need:

```bash
# Install Copilot CLI (if not already installed)
npm install -g @github/copilot-cli

# Authenticate
copilot auth login
```

---

## Troubleshooting

### "Server unreachable"

```
❌ Error: Unable to connect to server01
```

**Check:**
1. Server hostname/IP is correct
2. Server is online (`ping server01`)
3. PowerShell remoting is enabled on target: `Enable-PSRemoting -Force && winrm quickconfig`
4. Firewall allows port 5986 (HTTPS)
5. Network connectivity between your machine and server exists

### "Access denied"

```
❌ Error: Access denied on server01
```

**Check:**
1. Your user is admin on the target server
2. Credentials are correct (if using explicit mode)
3. User is in the Administrators group on target
4. Account is not disabled or locked

### "WinRM not responding"

```
❌ Error: WinRM is not responding
```

**On the target server:**
```powershell
# Check WinRM service is running
Get-Service WinRM

# Start it if needed
Start-Service WinRM

# Re-enable PowerShell remoting
Enable-PSRemoting -Force
```

### "Command timed out"

If diagnostics hang or timeout:
1. Target server may be under heavy load (memory/CPU maxed)
2. Network latency is high
3. Try a simpler diagnostic first (general health vs. deep analysis)
4. If consistently timing out, check target server is responsive to basic commands

---

## How It Works

### Architecture

```
User Question (Copilot CLI)
         ↓
Win-Investigator Agent
  • Parse question → Identify server & concern
  • Route to appropriate diagnostic skill
  • Connect via PowerShell remoting
         ↓
Diagnostic Skills (.squad/skills/)
  • overview — System info, baseline health
  • disk-storage — Drive capacity, cleanup candidates
  • memory-cpu — Resource usage, top processes
  • services-events — Service status, event logs
  • network — Connectivity, network config
         ↓
Data Collection (PowerShell + CIM)
  • Remote execution via Invoke-Command
  • CIM sessions for WMI data (faster, more reliable)
  • Structured object output (not raw text)
         ↓
Formatted Report (to Copilot CLI)
  • Prioritized findings
  • Status indicators (🟢 🟡 🔴)
  • Actionable next steps
         ↓
User
```

### Why PowerShell Remoting?

- **No agents to install** — Windows built-in (WinRM)
- **Secure** — Kerberos or TLS encryption by default
- **Fast** — Direct server connection, no API overhead
- **Rich data** — Access to WMI, performance counters, event logs, services, network config

---

## Output Format

All diagnostic reports follow a consistent structure:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 WIN-INVESTIGATOR REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SERVER: [hostname or IP]
STATUS: [🟢 Healthy | 🟡 Warning | 🔴 Critical]
TIMESTAMP: [ISO 8601, UTC timezone]

───────────────────────────────────────────────────
FINDINGS (in priority order, most critical first)
───────────────────────────────────────────────────

[Finding 1]
  Status: [Severity indicator]
  Details: [Specific measurements and data]
  Impact: [What does this mean for your business?]
  Action: [What you should do, if anything]

[Finding 2, 3, etc.]
  ...

───────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────

[1-2 sentence plain English summary of server health and next steps]
```

---

## Status Indicators

- **🟢 Healthy** — System is normal, no action required
- **🟡 Warning** — Trend is concerning or approaching a threshold; monitor or investigate
- **🔴 Critical** — Action required immediately, service may be impacted or at risk

---

## Limitations

**Win-Investigator is a diagnostic tool, not an automation tool.**

It can:
- ✅ Report what's happening on your servers
- ✅ Identify problems and suggest causes
- ✅ Recommend next steps
- ✅ Collect and prioritize findings

It cannot:
- ❌ Restart services or processes (escalate to admin)
- ❌ Modify server configuration (escalate to admin)
- ❌ Add disk space or expand volumes (escalate to infrastructure)
- ❌ Fix application-level issues (escalate to app owner/DBA)
- ❌ Resolve security incidents (escalate to security team)

**When to escalate:**

- On-call admin — Service restarts, configuration changes
- DBA — Database-specific issues (SQL Server, replication, backups)
- Security team — Security events, unauthorized access, malware
- Infrastructure — Capacity planning, disk expansion, hardware

---

## Examples

### Example: General Health Check

```bash
$ copilot "What's going on with server01?"

Parsing... server01, general health check, using current user credentials
Connecting to server01 via PowerShell remoting...
Running overview + key health checks...

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

server01 is mostly healthy but has two concerns: disk space is critically low at 92% full, 
and memory is elevated at 78%. Prioritize freeing disk space to prevent application failures, 
then monitor memory trends.

Next steps: Clean C:\Temp and C:\Windows\Temp directories, then review IIS logs for archival.
```

### Example: Specific Concern (Disk)

```bash
$ copilot "server01 is out of disk space. What can I delete?"

Parsing... server01, disk space concern, using current user credentials
Running disk-storage diagnostic skill...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 WIN-INVESTIGATOR REPORT — DISK ANALYSIS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SERVER: server01
FOCUS: Disk Space Analysis
STATUS: 🔴 Critical
TIMESTAMP: 2026-03-09T14:32:00Z

───────────────────────────────────────────────────
FINDINGS
───────────────────────────────────────────────────

💾 Volume C: at Capacity
  Total: 60 GB | Used: 55.4 GB | Free: 4.6 GB (92% full)
  Danger: Below 10% free threshold

📂 Top Folders (cleanup candidates, in order of size):
  1. C:\Windows\Logs — 8.2 GB
     Contents: Event logs, IIS logs, system logs
     Risk: Low (logs can be archived)

  2. C:\inetpub\logs\LogFiles — 6.1 GB
     Contents: Old IIS request logs (Jan-Feb 2026)
     Risk: Very low (old logs, safe to archive)

  3. C:\Temp — 2.8 GB
     Contents: Application temp files
     Risk: Very low (apps recreate as needed)

  4. C:\Windows\Temp — 1.9 GB
     Contents: System temp files
     Risk: Very low (system recreates)

  5. C:\Program Files\App\Backup — 1.7 GB
     Contents: Application backups (dates unknown)
     Risk: Medium (verify before deleting)

🗑️ Safe Cleanup Sequence:
  Step 1: Delete C:\Windows\Temp (1.9 GB freed, very safe)
  Step 2: Delete C:\Temp (2.8 GB freed, very safe)
  Step 3: Archive IIS logs to external storage (6.1 GB freed)
           → After archiving, delete: C:\inetpub\logs\LogFiles\*.log
  Step 4: Review backups in C:\Program Files\App\Backup (1.7 GB potential)
           → Verify these are outdated before deleting

───────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────

C: drive is critically full. You can safely free ~11.5 GB by cleaning temp directories 
and archiving old IIS logs. Start with the temp directories (lowest risk), then tackle IIS logs 
if needed. This should bring you to a healthier free space level.

Next steps: Execute the cleanup sequence above, then verify free space with a follow-up check.
```

---

## Contributing

Issues, feature requests, and pull requests are welcome. See `.github/CONTRIBUTING.md` for guidelines.

---

## License

See `LICENSE` file in this repository.

---

## Support

For questions or issues:

1. Check this README and the troubleshooting section
2. Review existing GitHub issues
3. Open a new issue with server details (hostname, OS version, error message)

---

## Architecture & Development

**For developers and squad members:**

- Agent instructions: `.github/copilot-instructions.md`
- Agent definition: `.github/agents/win-investigator.md`
- Diagnostic skills: `.squad/skills/` (PowerShell implementations)
- Team charter and roles: `.squad/agents/{name}/charter.md`
- Project decisions: `.squad/decisions.md`

See `.squad/team.md` for the team roster and member expertise.

---

**Built with ❤️ by the win-investigator squad.**

*Last updated: 2026-03-09*
