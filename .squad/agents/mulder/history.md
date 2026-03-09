# Project Context

- **Owner:** Anthony Watherston
- **Project:** win-investigator — AI-driven Windows Server troubleshooting via PowerShell remoting
- **Stack:** PowerShell, Windows Server, Copilot CLI (agents/skills/instructions)
- **Created:** 2026-03-09

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

### 2026-03-09 - Initial Skill Library Created

**Created comprehensive PowerShell diagnostic skills** under `skills/` directory (product files, not team .squad/skills):

1. **connectivity** - Test WinRM, establish sessions, handle credentials
2. **server-overview** - Hostname, OS, uptime, domain, hardware basics
3. **processes** - Running processes, CPU/memory usage, hung processes
4. **performance** - CPU, memory, disk I/O, performance counters with thresholds
5. **disk-storage** - Volume space, disk health, SMART data, large files
6. **services** - Service status, crashes, dependencies, management
7. **installed-apps** - Software inventory, recent installs, hotfixes
8. **network** - IP config, DNS, ports, firewall, connectivity tests
9. **roles-features** - Server roles, IIS/AD DS config, feature inventory
10. **event-logs** - Critical/error events, crashes, reboots, security

**Key patterns established:**
- Use `$ServerName` for target server variable
- Use `$Credential` for alternate credentials (null for current user)
- Prefer CIM over WMI: `Get-CimInstance` not `Get-WmiObject`
- All remote calls use `Invoke-Command` with `-ErrorAction Stop` and try/catch
- Return structured objects, not raw text
- Include interpretation tables, common issues, error handling examples
- Each skill is a complete reference document with multiple code patterns

**File locations:**
- Product skill files: `C:\Source\win-investigator\skills/*/SKILL.md`
- Each subdirectory contains one SKILL.md file with complete diagnostic instructions

**Technical decisions:**
- Skills are instruction files for AI agents, not executable PowerShell modules
- Each SKILL.md contains code blocks the agent should run, plus interpretation guidance
- Focus on comprehensive error handling and timeout patterns
- Include both quick checks and deep diagnostic variants

---

### Azure VM Connectivity Skill Created

**Date:** Session current  
**What:** Created `skills/azure-connectivity/SKILL.md` and updated connectivity skill + copilot-instructions for Azure VM public IP scenarios.

**Key patterns for Azure VM remoting:**
- Detect Azure targets: public IP (not RFC1918) or `*.cloudapp.azure.com` hostnames
- Always HTTPS (port 5986) — never HTTP over public internet
- Always explicit credentials — no Kerberos over public IP
- TrustedHosts must include target IP on client side
- NSG inbound rule for TCP 5986 is required in Azure
- `New-PSSessionOption -SkipCACheck -SkipCNCheck` for self-signed certs
- `New-CimSessionOption -UseSsl -SkipCACheck -SkipCNCheck` for CIM sessions
- Alternative approaches: Azure Bastion, Serial Console, Run Command

**Files created/updated:**
- `skills/azure-connectivity/SKILL.md` — New comprehensive skill (pre-flight detection, NSG checks, WinRM HTTPS setup, TrustedHosts, PSSession/CimSession over SSL, error table, alternatives)
- `skills/connectivity/SKILL.md` — Added Azure target detection, HTTPS connection path, Azure-specific errors
- `.github/copilot-instructions.md` — Added Azure VM credentials section, Azure-specific errors, HTTPS transport in workflow, azure-connectivity to skills reference

---

**Team synchronization after initial build:**

### Scully's Architectural Design
Scully completed skill-based modular architecture design with single orchestrator skill. Key contribution: established patterns for PowerShell remoting, credential handling (passed as parameters, never stored), and structured diagnostics returning PSObjects.

**Architectural decisions:**
- Functions accept `ComputerName` and `Credential` parameters
- Intelligence layer (Copilot agent) + Execution layer (PowerShell skill) separation
- Return structured PSObjects, not formatted text, for programmatic filtering
- Diagnostic areas: Processes, Performance, Disks, Services, Apps, Network, Roles

### Doggett's Documentation Framework
Doggett completed three-layer documentation with comprehensive copilot-instructions.md, agent definition, and user-facing README. Established patterns for Parse → Connect → Diagnose → Report workflow with status indicators and error handling.

### Readiness for Next Phase
All three agents completed work successfully. Mulder's skills are production-ready. Project ready for Skinner (testing) to write test scenarios and validate diagnostic functions.

---

### HTTPS-Only Connection Pattern Enforced

**Date:** Current session  
**Directive from:** Anthony Watherston (owner)

**What changed:** Swept ALL skill files, instruction files, agent definition, docs, and README to enforce a universal HTTPS-only connection pattern. Every `Invoke-Command`, `New-PSSession`, `New-CimSession`, and `Test-WSMan` call now uses:

- **HTTPS on port 5986** — the ONLY transport (HTTP/5985 removed entirely)
- **`-SkipCACheck -SkipCNCheck`** on all session options — handles self-signed certs and IP address connections
- **No TrustedHosts modification** — `-SkipCACheck` and `-SkipCNCheck` eliminate the need
- **IP addresses supported directly** — no DNS requirement, no TrustedHosts workaround

**Files updated (14 total):**
- `skills/connectivity/SKILL.md` — Complete rewrite. Removed Azure detection branching, HTTP path, TrustedHosts. HTTPS+SkipCA+SkipCN is the single universal pattern.
- `skills/azure-connectivity/SKILL.md` — Removed TrustedHosts step entirely. Simplified to Azure-specific concerns (NSG rules, WinRM HTTPS listener setup, alternative access methods).
- `skills/server-overview/SKILL.md` — All `Invoke-Command` and splatted `$invokeParams` updated with UseSSL/Port/SessionOption.
- `skills/processes/SKILL.md` — Same treatment.
- `skills/performance/SKILL.md` — Same treatment.
- `skills/disk-storage/SKILL.md` — Same treatment.
- `skills/services/SKILL.md` — Same treatment.
- `skills/installed-apps/SKILL.md` — Same treatment.
- `skills/network/SKILL.md` — Same treatment. Port table updated.
- `skills/roles-features/SKILL.md` — Same treatment.
- `skills/event-logs/SKILL.md` — Same treatment.
- `.github/copilot-instructions.md` — Updated connection workflow, error handling, skills reference. Removed TrustedHosts guidance.
- `.github/agents/win-investigator.md` — Added connection transport section. Updated error handling.
- `README.md` + `docs/*.md` — Updated all port references, firewall guidance, TrustedHosts sections.

**Standard splatted pattern for diagnostic skills:**
```powershell
$SessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
$invokeParams = @{
    ComputerName  = $ServerName
    UseSSL        = $true
    Port          = 5986
    SessionOption = $SessionOption
    ScriptBlock   = $scriptBlock
    ErrorAction   = 'Stop'
}
if ($Credential) { $invokeParams['Credential'] = $Credential }
$result = Invoke-Command @invokeParams
```

**Key learning:** A universal connection pattern (HTTPS everywhere, SkipCA/SkipCN, no TrustedHosts) simplifies the entire project dramatically. No more branching logic for Azure vs on-prem, no TrustedHosts management, IP addresses just work.
