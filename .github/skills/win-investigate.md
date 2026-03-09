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

- **Processes** — Top CPU/memory consumers, unusual processes
- **Performance** — CPU usage, memory pressure, disk I/O
- **Disks** — Free space, capacity, volumes
- **Services** — Stopped services that should be running, recent failures
- **Installed Apps** — Recently installed software, version info
- **Network Config** — IP configuration, DNS settings, connectivity
- **Roles & Features** — Installed Windows roles/features

## PowerShell Remoting Approach

```powershell
# Connection pattern
$session = New-PSSession -ComputerName $serverName -Credential $credential -ErrorAction Stop

# Invoke diagnostics
$results = Invoke-Command -Session $session -ScriptBlock {
    # Collect diagnostic data
    # Return structured object
}

# Cleanup
Remove-PSSession -Session $session
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

- **Default:** Use current user credentials (no explicit credential object)
- **Explicit:** If user provides credentials, create PSCredential object:
  ```powershell
  $securePassword = ConvertTo-SecureString $password -AsPlainText -Force
  $credential = New-Object System.Management.Automation.PSCredential($username, $securePassword)
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
