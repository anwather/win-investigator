---
name: win-investigate
description: "Main orchestrator skill for Windows Server diagnostics via PowerShell remoting"
---

# win-investigate

**Main orchestrator skill for Windows Server diagnostics.**

## Purpose

This skill interprets user questions about Windows Server health and orchestrates the appropriate diagnostic checks. It determines which specific diagnostic areas to investigate and aggregates the results into a clear summary.

## When to Use

Invoke this skill when the user asks about:
- Server health or status
- Performance issues or slowness
- Disk space or storage
- Running processes or services
- Network configuration
- Installed applications or roles
- General "what's going on" questions

## How It Works

1. **Parse the question** — Extract server name(s) and diagnostic intent
2. **Determine scope** — Decide which diagnostic areas are relevant
3. **Collect credentials** — Use current user or prompt for explicit credentials if needed
4. **Execute diagnostics** — Invoke PowerShell remoting to collect data
5. **Aggregate results** — Combine data into a structured summary
6. **Present findings** — Format results as a clear diagnostic report

## Diagnostic Areas

The skill can investigate these areas (modular, can be invoked independently):

- **Overview** — OS version, uptime, hostname, hardware (FAST: 2-5s)
- **Processes** — Top CPU/memory consumers, unusual processes (MODERATE: 5-15s)
- **Performance** — CPU usage, memory pressure, disk I/O (MODERATE: 3-10s)
- **Disks** — Free space, capacity, volumes (FAST: 2-5s)
- **Services** — Stopped services that should be running, recent failures (MODERATE: 3-8s)
- **Network Config** — IP configuration, DNS settings, connectivity (MODERATE: 3-10s)
- **Event Logs** — Critical errors, warnings, crashes (SLOW: 15-60s)
- **Installed Apps** — Recently installed software, version info (SLOW: 5-10s registry, 30-120s Win32_Product)
- **Roles & Features** — Installed Windows roles/features (SLOW: 10-30s)

## Parallel Execution for Full Investigations

For full investigations (user asks "what's going on?" or "check everything"), run ALL diagnostics as background jobs in parallel:

**Benefits:**
- Reduces total wait time from 2-3 minutes (sequential) to ~30-60 seconds (parallel)
- User gets incremental progress updates as jobs complete
- Slow diagnostics (event logs, roles) don't block fast ones (overview, disks)

**Pattern:**
```powershell
# Launch all diagnostics as background jobs using -AsJob with Invoke-Command
$jobs = @{}
$jobs['Overview'] = Invoke-Command @connParams -AsJob -ScriptBlock { ... }
$jobs['Disks'] = Invoke-Command @connParams -AsJob -ScriptBlock { ... }
$jobs['Performance'] = Invoke-Command @connParams -AsJob -ScriptBlock { ... }
# ... etc

# Collect results as they complete with timeout
foreach ($name in $jobs.Keys) {
    $job = $jobs[$name]
    $completed = $job | Wait-Job -Timeout 120
    if ($completed) {
        $results[$name] = Receive-Job -Job $job
        Write-Host "  ✓ $name complete" -ForegroundColor Green
    }
}
```

**When to use parallel execution:**
- Full investigations (generic "what's going on?" questions)
- Multiple diagnostic areas requested at once (2+ areas)

**When NOT to use parallel execution:**
- Single specific concern (just check disk, just check one service)
- User asks about one specific metric

## PowerShell Remoting Approach

**Sequential (single diagnostic):**
```powershell
# One-shot connection for focused diagnostics
$invokeParams = @{
    ComputerName  = $ServerName
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    ScriptBlock   = { # Diagnostic code here }
    ErrorAction   = 'Stop'
}
if ($credential) { $invokeParams['Credential'] = $credential }
$result = Invoke-Command @invokeParams
```

**Parallel (full investigation):**
```powershell
# Establish connection params once
$connParams = @{
    ComputerName  = $ServerName
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    ErrorAction   = 'Stop'
}
if ($credential) { $connParams['Credential'] = $credential }

# Launch each diagnostic as a background job
$jobs = @{}
$jobs['Overview'] = Invoke-Command @connParams -AsJob -ScriptBlock { ... }
$jobs['Performance'] = Invoke-Command @connParams -AsJob -ScriptBlock { ... }
$jobs['Disks'] = Invoke-Command @connParams -AsJob -ScriptBlock { ... }
# ... etc

# Collect results as they complete
$results = @{}
foreach ($name in $jobs.Keys) {
    $job = $jobs[$name]
    $completed = $job | Wait-Job -Timeout 120
    if ($completed) {
        $results[$name] = Receive-Job -Job $job -ErrorAction Stop
    }
}
$jobs.Values | Remove-Job -Force -ErrorAction SilentlyContinue
```

## Error Handling

- Test connectivity before attempting diagnostics
- Handle credential failures gracefully
- Collect partial data if some diagnostics fail
- Report what succeeded and what failed
- Suggest remediation steps for common issues

## Output Format

Present results as:
1. **Summary** — Quick status overview (healthy/issues detected)
2. **Key Findings** — Top issues that need attention
3. **Detailed Data** — Organized by diagnostic area
4. **Recommendations** — Suggested next steps if issues found

## Credential Flow

- **Default:** Use current user credentials (no explicit credential object needed)
- **Explicit:** If user needs alternate credentials, check for pre-created `$credential` variable:
  ```powershell
  # Agent checks for variable (user must create BEFORE running gh copilot)
  if (-not $credential) {
      Write-Host "⚠️ I need credentials to connect to $ServerName." -ForegroundColor Yellow
      Write-Host "Please run this in your PowerShell session:" -ForegroundColor Cyan
      Write-Host "  `$credential = Get-Credential" -ForegroundColor White
      Write-Host "Then ask me again and I'll connect using those credentials." -ForegroundColor Cyan
      return
  }
  # Agent uses pre-created credential
  $session = New-PSSession -ComputerName $ServerName -Credential $credential -ErrorAction Stop
  ```

## Skill Implementation Notes

This is a meta-skill that coordinates other diagnostic functions. The actual PowerShell code should be modular:

```powershell
function Get-ServerProcessInfo { }
function Get-ServerDiskInfo { }
function Get-ServerServiceInfo { }
function Get-ServerPerformanceInfo { }
function Get-ServerNetworkInfo { }
function Get-ServerRoleInfo { }
function Get-ServerAppInfo { }

function Invoke-WinInvestigation {
    param(
        [string[]]$ComputerName,
        [PSCredential]$Credential,
        [string[]]$DiagnosticAreas = @('All')
    )
    # Orchestration logic here
}
```

Each diagnostic function should:
- Accept ComputerName and Credential parameters
- Return a structured object (not formatted text)
- Handle errors and return error state if needed
- Be testable independently
