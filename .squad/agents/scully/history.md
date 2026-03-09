# Project Context

- **Owner:** Anthony Watherston
- **Project:** win-investigator — AI-driven Windows Server troubleshooting via PowerShell remoting
- **Stack:** PowerShell, Windows Server, Copilot CLI (agents/skills/instructions)
- **Created:** 2026-03-09

## Learnings

<!-- Append new learnings below. Each entry is something lasting about the project. -->

### 2026-03-10: Architecture Design

**Decision:** Skill-based modular architecture for win-investigator

**Structure:**
- Single orchestrator skill (`win-investigate`) at `.github/skills/win-investigate.md`
- Modular diagnostic functions in `src/diagnostics/` (one per area)
- Main orchestrator at `src/Invoke-WinInvestigation.ps1`
- Instructions at `.github/copilot-instructions.md`

**Diagnostic Areas:** Processes, Performance, Disks, Services, Apps, Network, Roles

**Key Patterns:**
- Functions accept `ComputerName` and `Credential` parameters
- Return structured PSObjects (not formatted text)
- Credential flow: passed as parameters, never stored
- Intelligence layer: Copilot agent interprets questions and decides what to run
- Execution layer: Skill provides PowerShell remoting and data collection

**Rationale:**
- Skills are native abstraction for Copilot CLI tools (not agents - those are for stateful workflows)
- Modularity enables independent development and testing of diagnostic areas
- Structured data enables programmatic filtering and multiple output formats
- Separation of intelligence (agent) and execution (skill) keeps concerns clean

**File Paths:**
- Architecture decision: `.squad/decisions/inbox/scully-architecture.md`
- Main skill: `.github/skills/win-investigate.md`
- Diagnostics: `src/diagnostics/Get-Server*.ps1`
- Orchestrator: `src/Invoke-WinInvestigation.ps1`

**Open Questions:**
- Concurrent vs sequential server processing?
- PSSession reuse across questions vs create/destroy per request?
- Default output format preference?

---

## Cross-Agent Context (2026-03-10)

**Team synchronization after initial build:**

### Mulder's Diagnostic Skills
Mulder successfully built 10 diagnostic skill files under `skills/` implementing comprehensive PowerShell-based Windows Server diagnostics. Key patterns include:
- Use `$ServerName` for target server variable
- Use `$Credential` for alternate credentials (null for current user)
- Prefer CIM over WMI: `Get-CimInstance` not `Get-WmiObject`
- All remote calls use `Invoke-Command` with `-ErrorAction Stop` and try/catch
- Return structured objects, not raw text
- Include interpretation tables and error handling examples
- Each skill is a complete reference document with multiple code patterns

**Skills created:** connectivity, server-overview, processes, performance, disk-storage, services, installed-apps, network, roles-features, event-logs

**Product location:** `C:\Source\win-investigator\skills/*/SKILL.md`

### Doggett's Documentation Framework
Doggett completed three-layer documentation structure:
- `.github/copilot-instructions.md` — Comprehensive agent behavior reference with parsing logic, workflow, and troubleshooting
- `.github/agents/win-investigator.md` — Shorter reference card for the agent
- `README.md` — User-facing documentation with prerequisites, examples, and troubleshooting

**Key patterns:** Parse → Connect → Diagnose → Report, with status indicators (🟢 🟡 🔴), error handling with suggested next steps, and escalation guidance.

### Readiness for Next Phase
All three agents completed their work successfully. Project is now ready for Skinner (testing) to write test scenarios for each diagnostic area and validate both current-user and explicit credential flows.
