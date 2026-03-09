---
updated_at: 2026-03-10T09:44:00.000Z
focus_area: Architecture and Foundation
active_issues: []
---

# What We're Focused On

**Current Sprint:** Architecture and Foundation  
**Goal:** Establish the diagnostic framework with skill-based modular design

## This Week

### Architecture ✅ COMPLETE
- Scully designed skill-based architecture
- Created directory structure and placeholder files
- Documented architectural decisions in `.squad/decisions/inbox/scully-architecture.md`

### Implementation 🔄 IN PROGRESS
**Mulder (Diagnostics Engineer):**
- Implement diagnostic functions in `src/diagnostics/`
- Start with: Process and Disk diagnostics
- Then: Services, Performance, Network, Roles, Apps

**Doggett (Documentation):**
- Write user-facing documentation and usage examples
- Document credential handling patterns
- Common troubleshooting scenarios

**Skinner (Tester):**
- Design test scenarios for each diagnostic area
- Test current-user vs explicit credentials
- Test error handling and partial failures

## Key Decisions Made

1. **Skill-based architecture** — Single orchestrator skill coordinates modular functions
2. **Structured data flow** — Functions return PSObjects for programmatic use
3. **Credential passing** — Flows as parameters, never stored
4. **Intelligence via Copilot agent** — Agent interprets questions and decides what to run

## Next Milestone

Complete core diagnostic functions (Processes, Disks, Services) and validate architecture with working implementation.
