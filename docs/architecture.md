---
layout: default
title: Architecture
nav_order: 6
description: "How Win-Investigator works — architecture, skill pipeline, and design decisions"
---

# Architecture
{: .no_toc }

How Win-Investigator works under the hood — for contributors and the curious.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

Win-Investigator is a Copilot CLI agent that uses a **skill-based modular architecture** with a single orchestrator. The intelligence layer (Copilot agent) interprets user questions and selects diagnostics, while the execution layer (PowerShell skills) collects data from remote servers.

```
┌─────────────────────────────────────────────────┐
│  User Question (Copilot CLI)                    │
│  "What is going on with server01?"              │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│  Win-Investigator Agent                         │
│  • Parse question → Identify server & concern   │
│  • Route to appropriate diagnostic skill        │
│  • Handle credentials (current user / explicit) │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│  Diagnostic Skills                              │
│  ┌────────────┐ ┌────────────┐ ┌─────────────┐ │
│  │connectivity│ │  overview   │ │disk-storage │ │
│  └────────────┘ └────────────┘ └─────────────┘ │
│  ┌────────────┐ ┌────────────┐ ┌─────────────┐ │
│  │performance │ │ processes  │ │  services   │ │
│  └────────────┘ └────────────┘ └─────────────┘ │
│  ┌────────────┐ ┌────────────┐ ┌─────────────┐ │
│  │  network   │ │ event-logs │ │installed-app│ │
│  └────────────┘ └────────────┘ └─────────────┘ │
│  ┌────────────┐                                 │
│  │roles-feat. │                                 │
│  └────────────┘                                 │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│  Data Collection (PowerShell + CIM)             │
│  • Remote execution via Invoke-Command          │
│  • CIM sessions for WMI data                    │
│  • Structured PSObject output                   │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│  Formatted Report                               │
│  • Prioritized findings (🟢 🟡 🔴)              │
│  • Actionable next steps                        │
│  • Plain English summary                        │
└─────────────────────────────────────────────────┘
```

---

## Design Principles

### Intelligence vs. Execution

Win-Investigator separates concerns into two layers:

| Layer | Responsibility | Technology |
|-------|---------------|------------|
| **Intelligence** | Parse questions, select diagnostics, format reports, suggest actions | Copilot agent (LLM) |
| **Execution** | Collect data from servers, run PowerShell commands, return structured results | PowerShell skills |

This separation means:
- The agent doesn't need to know PowerShell internals
- Skills don't need to understand natural language
- Each can be updated independently

### Skill-Based Modularity

Each diagnostic area is an independent skill with:
- A `SKILL.md` file documenting purpose, code, and interpretation guidance
- PowerShell code patterns for data collection
- Structured PSObject return values (not formatted text)
- Error handling and fallback logic

### Credentials as Pre-created Variables

Credentials are **never created inline** by the agent. The user creates a `$credential` variable
in their PowerShell session before starting `gh copilot`, and the agent uses it if present:

```
User creates $credential in PowerShell session:
  → $credential = Get-Credential (opens secure Windows login dialog)
    → User enters username/password in GUI dialog (NOT in chat)
      → User starts gh copilot
        → Agent checks: if (-not $credential) { tell user to create it }
          → Agent uses: if ($credential) { $params['Credential'] = $credential }
            → Skill uses for Invoke-Command -Credential
              → Credential persists for the PowerShell session lifetime
```

**Security principles:**
- Passwords are NEVER typed in the Copilot CLI chat
- The agent NEVER runs Get-Credential inline — the user creates it beforehand
- `$credential = Get-Credential` opens a secure Windows GUI dialog
- Passwords are never visible in conversation history
- PSCredential objects are never logged or displayed
- If `$credential` is not set, the agent tells the user:
  "Please run this in your PowerShell session: `$credential = Get-Credential`"

---

## Skill Pipeline

When a user asks a question, the pipeline flows through these stages:

### 1. Parse

The agent analyzes the question to extract:
- **Target server** — hostname or IP address
- **Concern area** — disk, memory, services, network, general, etc.
- **Urgency signals** — "critical", "down", "not working", "slow"
- **Credential mode** — current user (default) or pre-created `$credential` variable

### 2. Connect

The agent establishes a connection:
- Tests connectivity (ping + WinRM)
- Authenticates with current user or pre-created `$credential` variable
- Handles connection errors with specific guidance

### 3. Diagnose

The agent routes to one or more diagnostic skills:

| Concern | Skills Used |
|---------|-------------|
| General health | server-overview + performance + services |
| Disk space | disk-storage |
| Slow performance | performance + processes |
| Service issue | services + event-logs |
| Network problem | connectivity + network |
| Application issue | processes + installed-apps + event-logs |

### 4. Report

Results are formatted into the standard report structure:
- Header (server, status, timestamp)
- Findings (priority-ordered, with severity indicators)
- Summary (plain English, 1-2 sentences)
- Next steps (actionable recommendations)

---

## Skill Structure

Each skill lives in `skills/{skill-name}/` and contains a `SKILL.md` file with:

```
skills/
├── connectivity/
│   └── SKILL.md
├── server-overview/
│   └── SKILL.md
├── disk-storage/
│   └── SKILL.md
├── performance/
│   └── SKILL.md
├── processes/
│   └── SKILL.md
├── services/
│   └── SKILL.md
├── network/
│   └── SKILL.md
├── event-logs/
│   └── SKILL.md
├── installed-apps/
│   └── SKILL.md
└── roles-features/
    └── SKILL.md
```

### SKILL.md Anatomy

Each skill file follows a consistent structure:

1. **Purpose** — What the skill does
2. **PowerShell Code** — Actual code patterns the agent can use
3. **Interpreting Results** — Thresholds, meanings, what's normal vs. abnormal
4. **Common Issues** — Known problems and their causes
5. **Error Handling** — Fallback strategies when things go wrong
6. **Next Steps** — What to investigate based on findings

### Code Patterns

All skills follow these conventions:

```powershell
# Standard parameters
$ServerName = "TARGET_SERVER"
$Credential = $null  # Set if explicit credentials needed

# Remote execution pattern
$scriptBlock = {
    try {
        # Collect data using CIM (preferred over WMI)
        $data = Get-CimInstance -ClassName Win32_OperatingSystem

        # Return structured PSObject
        [PSCustomObject]@{
            Property1 = $data.SomeValue
            Property2 = $data.OtherValue
            Timestamp = Get-Date
        }
    } catch {
        throw "Diagnostic failed: $($_.Exception.Message)"
    }
}

# Execute remotely
$invokeParams = @{
    ComputerName = $ServerName
    ScriptBlock  = $scriptBlock
    ErrorAction  = 'Stop'
}
if ($Credential) { $invokeParams['Credential'] = $Credential }

$result = Invoke-Command @invokeParams
```

Key conventions:
- **CIM over WMI** — `Get-CimInstance` not `Get-WmiObject`
- **Structured returns** — PSObjects, not formatted text
- **Error handling** — try/catch with meaningful error messages
- **Credential flow** — Optional `$credential` parameter; user creates it before running `gh copilot`, null for current user. Agent NEVER runs Get-Credential inline.

---

## Why PowerShell Remoting?

Win-Investigator uses PowerShell remoting (WinRM) because:

| Benefit | Detail |
|---------|--------|
| **No agents to install** | WinRM is built into Windows |
| **Secure by default** | Kerberos or TLS encryption |
| **Fast** | Direct server connection, no API overhead |
| **Rich data** | Access to WMI, performance counters, event logs, services, network config |
| **Standard** | Widely used in enterprise Windows environments |

---

## Project Structure

```
win-investigator/
├── .github/
│   ├── agents/               # Agent definition files
│   ├── copilot-instructions.md  # Main agent instructions
│   └── workflows/            # GitHub Actions (CI, Pages)
├── skills/                   # Diagnostic skill definitions
│   ├── connectivity/
│   ├── server-overview/
│   ├── disk-storage/
│   ├── performance/
│   ├── processes/
│   ├── services/
│   ├── network/
│   ├── event-logs/
│   ├── installed-apps/
│   └── roles-features/
├── src/                      # Source code (orchestrator, diagnostics)
├── docs/                     # This documentation site
└── README.md                 # User-facing project README
```

---

## Contributing

Win-Investigator welcomes contributions. Key areas:

- **New diagnostic skills** — Add a new `skills/{name}/SKILL.md` with code patterns and interpretation guidance
- **Improve existing skills** — Add edge cases, better error handling, new checks
- **Documentation** — Improve this docs site, add examples, fix typos
- **Testing** — Validate diagnostics against real Windows Server environments

See the [GitHub repository](https://github.com/anwather/win-investigator) for issues and pull requests.

---

_Built by the Win-Investigator team._
