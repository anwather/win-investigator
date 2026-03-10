# Project Context

- **Owner:** Anthony Watherston
- **Project:** win-investigator — AI-driven Windows Server troubleshooting via PowerShell remoting
- **Stack:** PowerShell, Windows Server, Copilot CLI (agents/skills/instructions)
- **Created:** 2026-03-09

## Learnings

### Documentation Structure & Copilot Instructions (2026-03-09)

**Key patterns for agent instruction files:**
- `.github/copilot-instructions.md` is the main agent instruction file. It should be comprehensive, focused, and answer "what to do when" questions before they're asked.
- Agent definitions in `.github/agents/{agent-name}.md` are shorter (reference cards), point to the full instructions, and show example interactions.
- README.md is user-facing: it explains prerequisites, usage examples, and troubleshooting. It should assume the reader has never used this tool before.

**Win-Investigator instruction priorities:**
1. **Parse the question** — Identify target server, concern area, urgency signals
2. **Connect gracefully** — Handle default (current user) and explicit credential flows
3. **Run focused diagnostics** — Match concern to skill (overview, disk, memory, services, network)
4. **Report clearly** — Structured format with severity indicators (🟢 🟡 🔴), specific data, actionable next steps
5. **Handle errors** — Connection failures, access denied, WinRM not responding

**Output format (consistent across all diagnostic reports):**
- Header with server name, status indicator, timestamp
- Findings section (priority order, most critical first)
- Summary section (1-2 sentences plain English)
- Next steps (actionable escalation or investigation hints)

**Skills integration:**
- Skills live in `.squad/skills/` and implement domain-specific logic (PowerShell, CIM, data collection)
- Agent instructions reference skills but don't implement them (Mulder's domain)
- Skills include: overview, disk-storage, memory-cpu, services-events, network, general-health

**Credential handling pattern:**
- Default: current user (implicit, no prompting)
- Explicit: user specifies "with domain\admin credentials" → agent prompts for password
- Error messages should suggest next steps (enable WinRM, verify firewall, check admin rights)

---

## Cross-Agent Context (2026-03-10)

**Team synchronization after initial build:**

### Scully's Architectural Design
Scully completed skill-based modular architecture with single orchestrator skill. Established patterns for PowerShell remoting, credential handling (passed as parameters, never stored), and structured diagnostics. Created architectural decision document outlining diagnostic areas, directory structure, and intelligence vs. execution layer separation.

**Key contributions to Doggett's work:**
- Defined credential flow patterns that documentation must explain
- Established structured data patterns (PSObjects) that docs should reference
- Created open questions about concurrent processing and session reuse for future iterations

### Mulder's Diagnostic Skills
Mulder created 10 diagnostic skill files with comprehensive PowerShell code patterns and interpretation guidance. Key patterns:
- Use `$ServerName` for target server variable
- Use `$Credential` for alternate credentials (null for current user)
- Prefer CIM over WMI: `Get-CimInstance` not `Get-WmiObject`
- All remote calls use `Invoke-Command` with `-ErrorAction Stop` and try/catch
- Return structured objects, not raw text

**Integration point:** Skills referenced in Doggett's instructions (overview, disk-storage, memory-cpu, services-events, network, general-health)

### Readiness for Next Phase
All three core agents completed work successfully. Architecture is stable, diagnostics are implemented, documentation is comprehensive. Project ready for Skinner (testing) to write test scenarios and validate against real Windows Server environments.

### GitHub Pages Documentation Site (2026-03-10)

**What was built:**
- Full Jekyll documentation site using `just-the-docs` theme (dark mode) in `docs/` folder
- 7 content pages: Home (index), Getting Started, Usage Guide, Diagnostics Reference, Examples, Troubleshooting, Architecture
- Jekyll config with search enabled, dark color scheme, back-to-top, GitHub aux links
- Gemfile and .gitignore for local Jekyll development
- GitHub Actions workflow (`.github/workflows/pages.yml`) for automatic deployment from `main` branch

**Content strategy:**
- Redistributed README.md content across purpose-built pages (not copy-paste — restructured for web)
- Pulled skill details from all 10 `skills/*/SKILL.md` files for the Diagnostics Reference page
- Used just-the-docs callout syntax (`{: .note }`, `{: .warning }`, `{: .important }`) for emphasis
- Consistent footer on every page: "Built by the Win-Investigator team"

**Key decisions:**
- Used `remote_theme` instead of gem theme for GitHub Pages compatibility
- Dark color scheme matches the terminal/CLI nature of the tool
- Pages workflow triggers only on `docs/**` changes to avoid unnecessary builds
- Diagnostics page serves as both a reference and a "what can I ask?" guide

---

### GitHub Pages Deployment (2026-03-09T2253)

**Outcome:** SUCCESS

Site is live and deployed. All 7 documentation pages are publicly accessible via GitHub Pages. Jekyll theme is configured with dark mode and search enabled. GitHub Actions workflow is triggering on updates to the `docs/` folder.

---

### Complete Beginner Documentation Rewrite (2026-03-10T1430)

**Requested by:** Anthony Watherston  
**Priority:** Complete rewrite for zero-knowledge audience  
**Outcome:** SUCCESS

**Files updated:**

1. **README.md** — Rewritten Quick Start section with hand-holding through installation:
   - Added "What You'll Need" checklist (Windows machine, admin access, internet, GitHub account)
   - Step-by-step walkthrough: GitHub CLI install → Authentication → Copilot CLI extension → Clone repo → First investigation
   - "How It Works (Simply)" section explaining the flow in plain English
   - Credentials section clarified for beginners
   - Kept all existing content (diagnostics table, examples, troubleshooting)

2. **docs/index.md** — Updated landing page:
   - Hero emphasizes simplicity: "Ask questions about your Windows Servers in plain English"
   - Added "Get Started in 5 Minutes" CTA button linking to getting-started.md
   - Added visual value prop: "Old Way" (manual commands) vs "Win-Investigator Way" (30-second diagnosis)

3. **docs/getting-started.md** — Complete rewrite as KEY onboarding page:
   - "What You'll Need" checklist upfront
   - Step 0-5 walkthrough with EXACT commands and expected output
   - "Verify Your Setup" section with 5 checkpoints
   - "Your First Investigation" with example command and full sample output
   - Comprehensive troubleshooting for first-time issues

4. **docs/usage.md** — Added beginner context:
   - "How to Talk to the Agent" — explains it's like chatting
   - "Simple Examples (Start Here)" with 5 concrete examples
   - "Tips for Better Questions" table with do/don't patterns
   - "Common Beginner Questions" FAQ section (6 questions)
   - Simplified output explanation with visual breakdown

5. **docs/troubleshooting.md** — Reorganized with beginner issues first:
   - "First Time Setup Issues" at top: gh not found, not authenticated, copilot extension not installed
   - "Connection Issues": Test-WSMan fails, Cannot resolve hostname, Access denied, Certificate invalid
   - "After Connection Issues" with full checklist
   - Advanced issues kept but lower priority

**Key patterns applied:**
- Every instruction is testable with expected output shown
- Exact commands provided as copy/paste blocks
- "What You'll Need" checklists reduce decision fatigue
- Beginner troubleshooting moved to top of troubleshooting.md
- FAQ section in usage.md answers common hesitations
- Connection pattern reminder embedded (HTTPS port 5986, -SkipCACheck -SkipCNCheck)
- Use of just-the-docs callouts for emphasis

**Tone:** Conversational, empathetic to non-technical users. Assumes zero knowledge of Copilot CLI, PowerShell remoting, or GitHub CLI.
