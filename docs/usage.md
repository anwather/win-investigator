---
layout: default
title: Usage Guide
nav_order: 2
description: "How to talk to Win-Investigator. Simple examples, credential handling, understanding reports."
---

# Usage Guide
{: .no_toc }

**How to ask Win-Investigator questions and read the reports.**
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## How to Talk to the Agent

Win-Investigator is like chatting with a colleague. Just describe what you want to know.

{: .note }
> **Include the server name** and **what you're concerned about**. The more specific, the better.

### Start an Interactive Session

```bash
cd win-investigator
gh copilot
```

You'll see a prompt asking what you need help with. Just type your question naturally.

---

## Simple Examples (Start Here)

### ✅ General Health Check

Ask this when you want a quick overview:

```
? "What is going on with server01?"
```

You'll get: OS info, uptime, disk space, memory, and a snapshot of services.

### ✅ Something Specific is Broken

Ask this when you know what's wrong:

```
? "server01 is running out of disk space"
```

You'll get: Detailed disk analysis, biggest folders, cleanup suggestions.

### ✅ Is a Service Running?

Ask about a specific service:

```
? "Is the SQL service running on server02?"
```

You'll get: Service status, startup type, recent errors, event log entries.

### ✅ Server is Slow

Ask about performance:

```
? "server03 is slow — what's using the CPU?"
```

You'll get: Top processes by CPU/memory, system load, baseline comparison.

### ✅ Can You Reach It?

Ask about network connectivity:

```
? "Can you reach server04?"
```

You'll get: Network adapter status, IP config, ping tests, open ports.

---

## Tips for Better Questions

| Do This | Why | Example |
|---------|-----|---------|
| **Include server name** | Agent needs to know which server | ✅ "server01 is slow" |
| **Include what's wrong** | Helps agent pick the right diagnostic | ✅ "disk is full" vs ❌ "server01" |
| **Use plain English** | No special syntax needed | ✅ "Why is disk full?" vs ❌ "Get-Volume -Size" |
| **Mention urgency if high** | Agent prioritizes critical issues | ✅ "server01 is down" vs "investigate server01" |

---

## Credentials: Default vs. Explicit

### Default (Uses Your Current User)

By default, Win-Investigator uses your Windows user account. If you have admin access to the target server, just ask:

```
? "Check server01"
```

No password prompt. It just works.

### Explicit Credentials (Different User)

If you need to use a different account (e.g., a dedicated admin):

```
? "Check server01 with domain\admin credentials"
```

Win-Investigator will prompt:

```
Enter password for domain\admin:
```

Type the password (it won't echo on screen) and press Enter.

**Important:** Credentials are used only for that one check. They are **never stored or reused**.

---

## Understanding the Report

All reports follow the same structure, so they're easy to read.

### The Parts of a Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 WIN-INVESTIGATOR REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SERVER: server01                          ← Target server
STATUS: 🟡 Warning                        ← Overall health
TIMESTAMP: 2026-03-09T14:30:00Z           ← When the check ran

───────────────────────────────────────────────────
FINDINGS (most critical first)
───────────────────────────────────────────────────

🔴 Disk Space Critical                    ← Severity indicator
  C: drive is 92% full (4.6 GB free)      ← Specific data
  Impact: Apps may fail...                ← Why this matters
  Action: Delete temp files...            ← What to do

🟡 Memory Usage Elevated
  Currently 78% (12.5 GB of 16 GB)
  Impact: Performance may degrade...
  Action: Monitor trends...

───────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────

server01 is mostly healthy but disk is critically low...  ← Plain English
Next steps: Clean temp, then monitor memory.             ← What to do next
```

### Reading the Status Indicators

| Icon | Meaning | What to Do |
|------|---------|-----------|
| 🟢 Healthy | No problems, everything is normal | Monitor as usual |
| 🟡 Warning | Concerning trend, approaching limit | Investigate or monitor closely |
| 🔴 Critical | Problem now, action required | Act immediately |

---

## Common Beginner Questions

### Q: Can I ask multiple questions in one session?

**A:** Yes! After you get a response, you'll see the prompt again. Just ask another question.

```
? "Check server01"
[gets report]
? "Now check server02"
[gets report for server02]
```

### Q: What if the agent doesn't understand my question?

**A:** Rephrase it to be more specific. Include the server name and what you're concerned about.

```
❌ "diagnose"              ← Too vague
✅ "server01 is slow"      ← Clear and specific
```

### Q: Can I use this on my local machine or must it be a server?

**A:** Any Windows machine with PowerShell remoting enabled. Works for local workstations too.

### Q: What if I don't have admin access to a server?

**A:** Use explicit credentials to log in as a user who does:

```
? "Check server01 with domain\admin credentials"
```

### Q: How long do checks usually take?

**A:** Most checks complete in 10-30 seconds. Some (like large disk scans) may take 1-2 minutes.

### Q: Can I save the report?

**A:** Copy/paste the report text into a document, email, or Slack. Reports are in plain text.

---

## Limitations & When to Escalate

{: .note }
> Win-Investigator is a **diagnostic tool**, not an automation tool. It reports findings and suggests actions — it does not make changes to your servers.

### What Win-Investigator Can Do

- ✅ Report what's happening on your servers
- ✅ Identify problems and suggest causes
- ✅ Recommend next steps
- ✅ Collect findings in priority order

### What It Cannot Do

- ❌ Restart services (escalate to on-call admin)
- ❌ Modify server configuration (escalate to on-call admin)
- ❌ Add disk space or expand volumes (escalate to infrastructure)
- ❌ Fix database issues (escalate to DBA)
- ❌ Resolve security incidents (escalate to security team)

### When to Escalate

**On-call admin:**
- Service restart needed
- Configuration change needed
- Immediate action required

**DBA:**
- Database-specific issues (SQL, replication, backups)
- Performance tuning needed

**Infrastructure:**
- Disk expansion or capacity planning
- Hardware issues (failed drives, RAM problems)

**Security Team:**
- Security events or unauthorized access
- Potential malware

---

_Built by the Win-Investigator team._
