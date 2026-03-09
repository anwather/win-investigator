# Squad Decisions

## Active Decisions

### 1. Architecture Decision: win-investigator Structure
**Date:** 2026-03-10  
**Architect:** Scully  
**Status:** Implemented

**Summary:** Skill-based modular architecture with single orchestrator skill (`win-investigate`). Diagnostic functions modularized under `src/diagnostics/`, each returning structured PSObjects. Credential flow: passed as parameters, never stored.

**Diagnostic Areas:** Processes, Performance, Disks, Services, Apps, Network, Roles

**Key Patterns:**
- Functions accept `ComputerName` and `Credential` parameters
- Return structured PSObjects (not formatted text)
- Intelligence layer (Copilot agent) + Execution layer (PowerShell skill)

**Alternatives Rejected:** Agent-based architecture (too stateful), instructions-only (no encapsulation), multiple individual skills (too granular), standalone module (not primary deliverable)

**Open Questions:**
- Concurrent vs sequential server processing?
- PSSession reuse across questions vs create/destroy per request?
- Default output format preference?

**Implications:**
- Mulder: Implement diagnostic functions independently with structured returns
- Doggett: Document skill invocation and user examples
- Skinner: Test each diagnostic area, credentials flows, partial failures
- Future: Easy to extend with new diagnostic areas or export as module

---

### 2. Decision: Copilot Instruction Framework for Win-Investigator
**Author:** Doggett  
**Date:** 2026-03-09  
**Status:** Implemented

**Summary:** Three-layer documentation structure established:

1. **`.github/copilot-instructions.md`** — Main agent instructions (comprehensive behavior reference)
2. **`.github/agents/win-investigator.md`** — Agent definition (reference card)
3. **`README.md`** — User-facing documentation (tutorial-style)

**Workflow:** Parse → Connect → Diagnose → Report

**Patterns Applied:**
- Status indicators: 🟢 🟡 🔴 for severity
- Error handling with suggested next steps for each error type
- Escalation guidance for when to stop and ask for help
- Skills integration: reference by name, don't implement

**Reasoning:** Separates concerns — instructions (agent behavior), definition (what agent is), docs (user interaction)

## Governance

- All meaningful changes require team consensus
- Document architectural decisions here
- Keep history focused on work, decisions focused on direction
