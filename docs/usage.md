---
layout: default
title: Usage Guide
nav_order: 2
description: "How to ask questions, handle credentials, and understand Win-Investigator output"
---

# Usage Guide
{: .no_toc }

Learn how to get the most out of Win-Investigator — from question patterns to reading reports.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Asking Questions

Win-Investigator understands natural language. Just describe what you want to know and mention the server name.

### Question Patterns

| Pattern | Example | What Happens |
|---------|---------|--------------|
| **General health** | _"What is going on with server01?"_ | Runs overview + key health checks |
| **Specific concern** | _"server01 is out of disk space"_ | Runs focused disk storage analysis |
| **Specific service** | _"Is SQL running on server02?"_ | Checks service status and related events |
| **Performance** | _"server03 is slow"_ | Runs CPU, memory, and process analysis |
| **Connectivity** | _"Can you reach server04?"_ | Tests network, ping, WinRM, ports |
| **Events** | _"Any errors on server01?"_ | Searches event logs for critical/error events |

### Tips for Better Questions

{: .note }
> Include the **server name** and your **concern** in the question. The more specific you are, the more focused the diagnostic will be.

```bash
# Good — specific server + specific concern
copilot "server01 is running out of disk space. What can I delete?"

# Good — specific server + specific service
copilot "Is the SQL service running on server02?"

# OK — general, triggers a broad health check
copilot "What is going on with server01?"
```

---

## Credential Handling

### Default: Current User

By default, Win-Investigator uses your current Windows user identity. No passwords to enter — it just works if you have network and admin access to the target server.

```bash
copilot "Check server01"
# Uses: your-domain\your-user (implicit)
```

### Explicit Credentials

If you need to use different credentials (e.g., a dedicated admin account):

```bash
copilot "Check server01 with domain\admin credentials"
# Prompts: Enter password for domain\admin
# Uses: domain\admin for this check
```

{: .warning }
> Credentials are used for the current check only. They are **never stored** — they are passed as parameters and discarded after the session.

---

## Understanding the Output

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

[Finding]
  Status: [Severity indicator]
  Details: [Specific measurements and data]
  Impact: [What does this mean for your business?]
  Action: [What you should do, if anything]

───────────────────────────────────────────────────
SUMMARY
───────────────────────────────────────────────────

[1-2 sentence plain English summary with next steps]
```

### Key Points

- **Findings are priority-ordered** — most critical issues appear first.
- **Each finding includes an action** — you always know what to do next.
- **The summary is human-readable** — share it directly with teammates or managers.

---

## Status Indicators

| Indicator | Meaning | Action |
|-----------|---------|--------|
| 🟢 **Healthy** | System is normal, no action required | Monitor as usual |
| 🟡 **Warning** | Trend is concerning or approaching a threshold | Monitor closely or investigate |
| 🔴 **Critical** | Action required immediately, service may be impacted | Act now |

These indicators appear at both the **report level** (overall server health) and the **finding level** (individual metrics).

---

## Limitations

Win-Investigator is a **diagnostic tool**, not an automation tool.

| It can… | It cannot… |
|---------|-----------|
| ✅ Report what's happening on your servers | ❌ Restart services or processes |
| ✅ Identify problems and suggest causes | ❌ Modify server configuration |
| ✅ Recommend next steps | ❌ Add disk space or expand volumes |
| ✅ Collect and prioritize findings | ❌ Fix application-level issues |

### When to Escalate

| Escalate To | When |
|-------------|------|
| **On-call admin** | Service restarts, configuration changes |
| **DBA** | Database-specific issues (SQL, replication, backups) |
| **Security team** | Security events, unauthorized access, malware |
| **Infrastructure** | Capacity planning, disk expansion, hardware |

---

_Built by the Win-Investigator team._
