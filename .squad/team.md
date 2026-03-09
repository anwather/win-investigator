# Squad Team

> win-investigator — AI-driven Windows Server troubleshooting service

## Coordinator

| Name | Role | Notes |
|------|------|-------|
| Squad | Coordinator | Routes work, enforces handoffs and reviewer gates. |

## Members

| Name | Role | Charter | Status |
|------|------|---------|--------|
| 🏗️ Scully | Lead | `.squad/agents/scully/charter.md` | ✅ Active |
| 🔧 Mulder | Backend Dev | `.squad/agents/mulder/charter.md` | ✅ Active |
| 🧪 Skinner | Tester | `.squad/agents/skinner/charter.md` | ✅ Active |
| 📝 Doggett | DevRel / Writer | `.squad/agents/doggett/charter.md` | ✅ Active |
| 📋 Scribe | Session Logger | `.squad/agents/scribe/charter.md` | ✅ Active |
| 🔄 Ralph | Work Monitor | — | 🔄 Monitor |

## Project Context

- **User:** Anthony Watherston
- **Project:** win-investigator — AI-driven service to troubleshoot Windows Servers via PowerShell remoting
- **Stack:** PowerShell, Windows Server, Copilot CLI (agents/skills/instructions)
- **Description:** Users ask questions like "What is going on with server01". The service investigates the server (processes, performance, disks, services, installed apps, network config, roles) and provides a diagnostic summary. Connections via PowerShell using current user credentials with option for alternate credentials.
- **Created:** 2026-03-09
