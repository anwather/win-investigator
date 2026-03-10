# Win-Investigator

**Ask questions about your Windows Servers in plain English. Get clear reports instantly.**

No complex commands. No scripts to learn. Just ask what's wrong, and get answers.

---

## 🚀 Quick Start — 5 Minutes

### What You'll Need

- ✅ A Windows machine (with PowerShell)
- ✅ Admin access to at least one Windows Server you want to investigate
- ✅ Internet connection (to authenticate with GitHub)

### Step 0: Install GitHub CLI

**Windows:** Open PowerShell and run:
```powershell
winget install GitHub.cli
```

**Or download:** https://cli.github.com/

Verify it installed:
```bash
gh --version
```

Expected: `gh version X.X.X (YYYY-MM-DD)`

### Step 1: Authenticate with GitHub

```bash
gh auth login
```

Follow the prompts:
- **What account do you want to log into?** → GitHub.com
- **What is your preferred protocol for Git operations?** → HTTPS
- **Authenticate Git with your GitHub credentials?** → Y
- **How would you like to authenticate GitHub CLI?** → Login with a web browser

A browser will open. Click **Authorize GitHub CLI** and you're done.

### Step 2: Install Copilot CLI Extension

```bash
gh extension install github/gh-copilot
```

Verify it's installed:
```bash
gh copilot --version
```

### Step 3: Clone win-investigator

```bash
gh repo clone anwather/win-investigator
cd win-investigator
```

**That's it!** All diagnostic skills are built-in and automatically loaded when you run `gh copilot` in this directory. No manual skill installation or configuration needed.

### Step 4: Start Your First Investigation

```bash
gh copilot
```

This opens an interactive session. You'll see a prompt — just ask:

```
? What would you like help with?

Ask anything about your Windows Servers, like:
  - "What is going on with server01?"
  - "Is the SQL service running on server02?"
  - "server03 is slow — what's using the CPU?"
```

Type your question and press Enter. The agent loads automatically with all diagnostic skills embedded—no installation or setup needed.

---

## How It Works (Simply)

```
You ask a question (plain English)
    ↓
Win-Investigator connects to your server (via PowerShell)
    ↓
Runs quick diagnostics (disk, memory, services, network)
    ↓
Gives you a clear report with priorities and next steps
```

---

## Example Questions You Can Ask

```bash
# General health
"What is going on with server01?"

# Specific concern
"server01 is running out of disk space. What can I delete?"

# Specific service
"Is the SQL service running on server02?"

# Performance
"server03 is slow — what's using the resources?"

# Network
"Can you reach server04?"
```

---

## Prerequisites Detail

### Windows PowerShell 5.1+ or PowerShell 7+

This comes built-in on Windows. Verify it:

```powershell
$PSVersionTable.PSVersion
```

Expected: `5.1` or higher (e.g., `7.4.0`)

### PowerShell Remoting on Target Servers

**One-time setup** (requires local admin on each target server):

```powershell
# RUN THIS ON EACH TARGET SERVER
Enable-PSRemoting -Force
winrm quickconfig -q
```

**Verify it works:**

```powershell
# Run from YOUR machine
Test-WSMan server01 -UseSSL -Port 5986 -SkipCACheck -SkipCNCheck

# Expected output:
# wsmid           : http://schemas.dmtf.org/wbem/wsman/identity/identity.xsd
# ProtocolVersion : http://schemas.dmtf.org/wbem/wsmanidentity/1.0.0
```

### Network Access to Target Servers

- Target server must be **reachable by hostname or IP address**
- Firewall must allow **port 5986** (HTTPS)
- If different networks: ensure **routing and firewall rules permit the connection**

### Admin Rights on Target Servers

- Your user must have **local administrator** rights on the target, OR
- You can provide **different credentials** when asking (see Usage section below)

### GitHub Account

You need a free GitHub account to authenticate. Sign up at https://github.com if you don't have one.

---

## Credentials

### Default (Automatic)

Win-Investigator uses **your current Windows user** by default. No passwords to enter.

```bash
gh copilot
? "Check server01"

# Uses: your-domain\your-user (automatically)
```

### Explicit Credentials (Secure Dialog)

If you need different credentials (e.g., a dedicated admin account), Win-Investigator will open 
a **secure Windows login dialog**:

```bash
gh copilot
? "Check server01 with domain\admin credentials"

# Windows opens a secure credential dialog (GUI)
# You enter username/password in the dialog, NOT in the chat
# Password is never visible in the conversation
```

**How the secure dialog works:**
1. Agent runs `Get-Credential`
2. Windows shows a login dialog box (GUI popup)
3. You type username and password in the dialog
4. Password is hidden and never appears in chat history

⚠️ **SECURITY: Never type passwords in the Copilot CLI chat.** Passwords typed in the conversation 
are visible in plain text. Always use the Get-Credential dialog for secure password entry.

**Important:** Credentials are **never stored**. They are used only for that check and then discarded.

### Pre-stored Credentials (Optional)

For frequently accessed servers, you can pre-store credentials using Windows Credential Manager:

```powershell
# One-time setup (run in PowerShell, outside Copilot):
Install-Module -Name CredentialManager -Force
New-StoredCredential -Target "server01" -UserName "domain\admin" -SecurePassword (Read-Host -AsSecureString)
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
