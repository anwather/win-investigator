---
layout: default
title: Home
nav_order: 0
description: "Win-Investigator — AI-driven Windows Server troubleshooting via Copilot CLI"
permalink: /
---

# Win-Investigator
{: .fs-9 }

AI-driven Windows Server troubleshooting via the Copilot CLI. Ask natural language questions about your Windows Servers and get clear, actionable diagnostic reports.
{: .fs-6 .fw-300 }

[Get Started](/win-investigator/getting-started){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/anwather/win-investigator){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What It Does

Win-Investigator diagnoses Windows Server issues by:

1. **Listening** to your question — _"What is going on with server01?"_
2. **Connecting** to the target server via PowerShell remoting
3. **Running focused diagnostics** based on your concern (disk, memory, services, network, and more)
4. **Reporting findings** in a clear, prioritized format with suggested next steps

No complex commands to memorize — just ask.

---

## Quick Example

```bash
copilot "What's going on with server01?"
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 WIN-INVESTIGATOR REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SERVER: server01
STATUS: 🟡 Warning

🔴 Disk Space Critical — C: is 92% full (4.6 GB free)
🟡 Memory Usage Elevated — 78% of 16 GB in use
🟢 Critical Services Running — SQL, IIS, Defender all healthy

Next steps: Clean temp folders, then monitor memory trends.
```

---

## Questions You Can Ask

- _"What is going on with server01?"_
- _"Why is server01 running out of disk space?"_
- _"Is the SQL service running on server02?"_
- _"server03 is slow — what's using the resources?"_
- _"Can you reach server04?"_

---

## 10 Built-In Diagnostics

Win-Investigator includes focused diagnostic skills for connectivity, server overview, disk storage, performance, processes, services, network configuration, event logs, installed applications, and Windows Server roles & features.

[See all diagnostics →](/win-investigator/diagnostics){: .btn .btn-outline }

---

{: .note }
> Win-Investigator is a **diagnostic tool**, not an automation tool. It reports findings and recommends actions — it does not make changes to your servers.

---

_Built by the Win-Investigator team._
