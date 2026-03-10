---
name: event-logs
description: "Analyze Windows Event Logs for errors, warnings, and critical events"
---

# Event Logs - System Event Analysis

## Purpose
Analyze Windows Event Logs to identify critical errors, warnings, and patterns that indicate system problems, crashes, or security issues.

## PowerShell Code

### Get Recent Critical and Error Events
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$DaysBack = 7
$MaxEvents = 50

$scriptBlock = {
    param($days, $maxEvents)
    
    try {
        $startDate = (Get-Date).AddDays(-$days)
        
        # Query System log for Critical and Error events
        $systemEvents = Get-WinEvent -FilterHashtable @{
            LogName   = 'System'
            Level     = 1, 2  # Critical = 1, Error = 2
            StartTime = $startDate
        } -MaxEvents $maxEvents -ErrorAction SilentlyContinue
        
        # Query Application log for Critical and Error events
        $appEvents = Get-WinEvent -FilterHashtable @{
            LogName   = 'Application'
            Level     = 1, 2
            StartTime = $startDate
        } -MaxEvents $maxEvents -ErrorAction SilentlyContinue
        
        # Combine and format
        $allEvents = ($systemEvents + $appEvents) | Sort-Object TimeCreated -Descending | Select-Object -First $maxEvents
        
        $eventData = foreach ($event in $allEvents) {
            [PSCustomObject]@{
                TimeCreated  = $event.TimeCreated
                LogName      = $event.LogName
                Level        = switch ($event.Level) {
                    1 { "Critical" }
                    2 { "Error" }
                    default { "Unknown" }
                }
                EventId      = $event.Id
                Source       = $event.ProviderName
                Message      = $event.Message.Split("`n")[0]  # First line only
            }
        }
        
        # Count events by source
        $eventGroups = $eventData | Group-Object Source | Sort-Object Count -Descending
        
        [PSCustomObject]@{
            TotalEvents      = $eventData.Count
            SystemEvents     = ($eventData | Where-Object { $_.LogName -eq "System" }).Count
            ApplicationEvents = ($eventData | Where-Object { $_.LogName -eq "Application" }).Count
            Events           = $eventData
            TopSources       = $eventGroups
            Timestamp        = Get-Date
        }
        
    } catch {
        throw "Event log analysis failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($DaysBack, $MaxEvents)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nAnalyzing event logs on $ServerName (last $DaysBack days)..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Event Log Summary ===" -ForegroundColor Yellow
    Write-Host "  Total Critical/Error Events: $($result.TotalEvents)"
    Write-Host "  System Log:                  $($result.SystemEvents)"
    Write-Host "  Application Log:             $($result.ApplicationEvents)"
    Write-Host ""
    
    if ($result.TotalEvents -eq 0) {
        Write-Host "✓ No critical or error events found in last $DaysBack days" -ForegroundColor Green
    } else {
        Write-Host "=== Top Event Sources ===" -ForegroundColor Yellow
        $result.TopSources | Select-Object -First 10 | Format-Table Count, Name -AutoSize
        
        Write-Host "`n=== Recent Critical/Error Events ===" -ForegroundColor Yellow
        $result.Events | Select-Object -First 30 | 
            Format-Table TimeCreated, Level, LogName, EventId, Source, @{L='Message';E={$_.Message.Substring(0, [Math]::Min(60, $_.Message.Length))}} -AutoSize
    }
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to analyze event logs on $ServerName : $($_.Exception.Message)"
}
```

### Check for Specific Event IDs
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$EventIds = @(1074, 1076, 6008)  # System shutdown events and unexpected reboots
$DaysBack = 30

$scriptBlock = {
    param($eventIds, $days)
    
    try {
        $startDate = (Get-Date).AddDays(-$days)
        
        $events = Get-WinEvent -FilterHashtable @{
            LogName   = 'System'
            Id        = $eventIds
            StartTime = $startDate
        } -ErrorAction SilentlyContinue
        
        $eventData = foreach ($event in $events) {
            [PSCustomObject]@{
                TimeCreated = $event.TimeCreated
                EventId     = $event.Id
                Level       = $event.LevelDisplayName
                Source      = $event.ProviderName
                Message     = $event.Message
            }
        }
        
        $eventData | Sort-Object TimeCreated -Descending
        
    } catch {
        @()  # Return empty if no events found
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($EventIds, $DaysBack)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nSearching for Event IDs: $($EventIds -join ', ') on $ServerName..." -ForegroundColor Cyan
    $events = Invoke-Command @invokeParams
    
    if ($events) {
        Write-Host "`nFound $($events.Count) matching event(s):" -ForegroundColor Yellow
        $events | Format-Table TimeCreated, EventId, Level, Source -AutoSize
        Write-Host ""
        
        # Show full message for first few events
        foreach ($event in ($events | Select-Object -First 3)) {
            Write-Host "Event $($event.EventId) at $($event.TimeCreated):" -ForegroundColor Cyan
            Write-Host $event.Message -ForegroundColor Gray
            Write-Host ""
        }
    } else {
        Write-Host "`n✓ No matching events found" -ForegroundColor Green
    }
    
    $events
    
} catch {
    Write-Error "Event search failed: $($_.Exception.Message)"
}
```

### Analyze System Crashes and Reboots
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$DaysBack = 30

$scriptBlock = {
    param($days)
    
    try {
        $startDate = (Get-Date).AddDays(-$days)
        
        # Event IDs related to crashes and reboots
        $crashEventIds = @(
            1001,  # BugCheck (Blue Screen)
            1074,  # System has been shutdown by a process/user
            1076,  # Follow-up to 1074 with shutdown reason
            6005,  # Event Log service started (boot)
            6006,  # Event Log service stopped (shutdown)
            6008,  # Unexpected shutdown
            6009,  # Processor info at boot
            41     # System rebooted without cleanly shutting down (Kernel-Power)
        )
        
        $events = Get-WinEvent -FilterHashtable @{
            LogName   = 'System'
            Id        = $crashEventIds
            StartTime = $startDate
        } -ErrorAction SilentlyContinue
        
        $eventData = foreach ($event in $events) {
            $eventType = switch ($event.Id) {
                1001 { "Blue Screen (BugCheck)" }
                1074 { "Planned Shutdown" }
                1076 { "Shutdown Reason" }
                6005 { "System Boot" }
                6006 { "System Shutdown" }
                6008 { "Unexpected Shutdown" }
                6009 { "System Boot Info" }
                41   { "Unexpected Reboot (Kernel-Power)" }
                default { "Other" }
            }
            
            [PSCustomObject]@{
                TimeCreated = $event.TimeCreated
                EventId     = $event.Id
                EventType   = $eventType
                Source      = $event.ProviderName
                Message     = $event.Message.Split("`n")[0..2] -join " "
            }
        }
        
        # Identify unexpected reboots
        $unexpectedReboots = $eventData | Where-Object { $_.EventId -in @(6008, 41, 1001) }
        $plannedShutdowns = $eventData | Where-Object { $_.EventId -in @(1074, 1076) }
        $bootEvents = $eventData | Where-Object { $_.EventId -eq 6005 }
        
        [PSCustomObject]@{
            AllEvents          = $eventData | Sort-Object TimeCreated -Descending
            UnexpectedReboots  = $unexpectedReboots
            PlannedShutdowns   = $plannedShutdowns
            BootCount          = $bootEvents.Count
        }
        
    } catch {
        throw "Crash analysis failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($DaysBack)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nAnalyzing system crashes and reboots on $ServerName (last $DaysBack days)..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Reboot/Crash Analysis ===" -ForegroundColor Yellow
    Write-Host "  Total Boot Events:       $($result.BootCount)"
    Write-Host "  Unexpected Reboots:      $($result.UnexpectedReboots.Count)" -ForegroundColor $(if ($result.UnexpectedReboots.Count -gt 0) { "Red" } else { "Green" })
    Write-Host "  Planned Shutdowns:       $($result.PlannedShutdowns.Count)"
    Write-Host ""
    
    if ($result.UnexpectedReboots) {
        Write-Host "⚠ UNEXPECTED REBOOTS DETECTED:" -ForegroundColor Red
        $result.UnexpectedReboots | Format-Table TimeCreated, EventType, Message -AutoSize
        Write-Host "  → Check for hardware issues, driver problems, or system crashes" -ForegroundColor Yellow
    } else {
        Write-Host "✓ No unexpected reboots detected" -ForegroundColor Green
    }
    
    if ($result.PlannedShutdowns) {
        Write-Host "`n=== Planned Shutdowns ===" -ForegroundColor Cyan
        $result.PlannedShutdowns | Select-Object -First 10 | 
            Format-Table TimeCreated, EventType, Message -AutoSize
    }
    
    $result
    
} catch {
    Write-Error "Failed to analyze crashes/reboots: $($_.Exception.Message)"
}
```

### Get Events by Source/Provider
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$ProviderName = "Microsoft-Windows-DistributedCOM"  # Example: DCOM errors
$DaysBack = 7
$MaxEvents = 30

$scriptBlock = {
    param($provider, $days, $max)
    
    try {
        $startDate = (Get-Date).AddDays(-$days)
        
        $events = Get-WinEvent -FilterHashtable @{
            ProviderName = $provider
            StartTime    = $startDate
        } -MaxEvents $max -ErrorAction Stop
        
        $events | ForEach-Object {
            [PSCustomObject]@{
                TimeCreated = $_.TimeCreated
                Level       = $_.LevelDisplayName
                EventId     = $_.Id
                Message     = $_.Message.Split("`n")[0]
            }
        } | Sort-Object TimeCreated -Descending
        
    } catch {
        throw "Event query failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($ProviderName, $DaysBack, $MaxEvents)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nRetrieving events from provider '$ProviderName'..." -ForegroundColor Cyan
    $events = Invoke-Command @invokeParams
    
    Write-Host "`nFound $($events.Count) event(s) from $ProviderName:" -ForegroundColor Yellow
    $events | Format-Table TimeCreated, Level, EventId, Message -AutoSize
    
    $events
    
} catch {
    Write-Error "Event query failed: $($_.Exception.Message)"
}
```

### Get Security Log Events (if accessible)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
# Usually requires elevated privileges
$DaysBack = 1
$MaxEvents = 100

$scriptBlock = {
    param($days, $max)
    
    try {
        $startDate = (Get-Date).AddDays(-$days)
        
        # Common security event IDs
        # 4624 = Successful logon
        # 4625 = Failed logon
        # 4634 = Logoff
        # 4648 = Logon using explicit credentials
        
        $events = Get-WinEvent -FilterHashtable @{
            LogName   = 'Security'
            Id        = @(4624, 4625, 4634, 4648)
            StartTime = $startDate
        } -MaxEvents $max -ErrorAction Stop
        
        $eventData = foreach ($event in $events) {
            $eventType = switch ($event.Id) {
                4624 { "Successful Logon" }
                4625 { "Failed Logon" }
                4634 { "Logoff" }
                4648 { "Explicit Credentials" }
                default { "Other" }
            }
            
            [PSCustomObject]@{
                TimeCreated = $event.TimeCreated
                EventId     = $event.Id
                EventType   = $eventType
                Level       = $event.LevelDisplayName
            }
        }
        
        # Count by type
        $eventGroups = $eventData | Group-Object EventType | Sort-Object Count -Descending
        
        [PSCustomObject]@{
            Events      = $eventData | Sort-Object TimeCreated -Descending
            Summary     = $eventGroups
        }
        
    } catch {
        throw "Security log query failed (may require admin privileges): $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($DaysBack, $MaxEvents)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nRetrieving security events from $ServerName (last $DaysBack day(s))..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Security Event Summary ===" -ForegroundColor Yellow
    $result.Summary | Format-Table Name, Count -AutoSize
    
    Write-Host "`n=== Recent Security Events ===" -ForegroundColor Yellow
    $result.Events | Select-Object -First 20 | 
        Format-Table TimeCreated, EventType, Level -AutoSize
    
    $result
    
} catch {
    Write-Error "Failed to retrieve security events: $($_.Exception.Message)"
    
    if ($_.Exception.Message -match "Access is denied") {
        Write-Host "  → Security log access requires administrator privileges" -ForegroundColor Yellow
    }
}
```

## Interpreting Results

### Event Levels

| Level | Severity | Meaning |
|-------|----------|---------|
| Critical (1) | Severe | System failure, data loss, critical service failure |
| Error (2) | Significant | Problem that needs attention, feature not working |
| Warning (3) | Potential issue | May lead to problems, informational |
| Information (4) | Normal | Normal operation, informational only |

### Common Critical Event IDs

| Event ID | Log | Meaning | Action |
|----------|-----|---------|--------|
| 1001 | System | Blue Screen (BugCheck) | Check hardware, drivers |
| 6008 | System | Unexpected shutdown | Power loss, hardware issue, crash |
| 41 | System | Kernel-Power unexpected reboot | Hardware issue, power problem |
| 7031 | System | Service crashed | Check service, application logs |
| 7034 | System | Service crashed unexpectedly | Investigate service failure |
| 10016 | System | DCOM permission error | Usually benign, can be ignored |

### System Event Patterns

**Frequent Reboots**
- Event 6005 (boot) multiple times per day → Instability, investigate cause

**Service Crashes**
- Events 7031/7034 → Service reliability issues, check application

**Disk Errors**
- Event 7 (disk bad block) → Failing disk, backup immediately

**Driver Issues**
- WER (Windows Error Reporting) events → Driver crashes, update drivers

### Application Event Patterns

**Application Crashes**
- Event 1000 (Application Error) → Application failure, check app logs

**.NET Errors**
- Event 1026 (.NET Runtime) → .NET application errors

**Windows Updates**
- Event 19 (Windows Update failure) → Update problems

### Security Event IDs

| Event ID | Meaning | Significance |
|----------|---------|--------------|
| 4624 | Successful logon | Normal activity |
| 4625 | Failed logon | Authentication failure, potential attack |
| 4634 | Logoff | Normal activity |
| 4648 | Logon with explicit credentials | RunAs, scheduled task |
| 4672 | Special privileges assigned | Admin logon |
| 4720 | User account created | Account management |

## Common Issues

### High Event Volume
- Repeating errors from same source → Configuration issue, bug
- Thousands of same event → Loop, runaway process

### Critical Events
- Blue Screen (1001) → Hardware, driver, or system corruption
- Unexpected shutdown (6008) → Power issue, hardware failure
- Disk errors → Failing storage

### Service Failures
- Service crashes (7031/7034) → Application bug, missing dependency
- Service timeout (7000/7009) → Service hanging, slow to start

## Error Handling

```powershell
# Handle missing or inaccessible event logs
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        $logs = @("System", "Application", "Security")
        
        foreach ($log in $logs) {
            try {
                $count = (Get-WinEvent -LogName $log -MaxEvents 1 -ErrorAction Stop).Count
                [PSCustomObject]@{
                    LogName    = $log
                    Accessible = $true
                    Error      = $null
                }
            } catch {
                [PSCustomObject]@{
                    LogName    = $log
                    Accessible = $false
                    Error      = $_.Exception.Message
                }
            }
        }
    } -ErrorAction Stop
    
    $result | Format-Table LogName, Accessible, Error -AutoSize
    
} catch {
    Write-Error "Event log check failed: $($_.Exception.Message)"
}
```

## Next Steps

Based on event log analysis:
- **Service crashes** → Check **services** for affected service, restart if needed
- **Unexpected reboots** → Check **disk-storage** health, hardware diagnostics
- **Application errors** → Check **processes** for problem application
- **DCOM errors** → Usually benign, can configure permissions if needed
- **High error count** → Prioritize by frequency and criticality
- **Security events** → Investigate failed logons, unusual account activity
