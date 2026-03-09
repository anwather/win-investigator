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

## Cross-Agent Context (2026-03-10)

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
