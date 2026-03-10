# Services - Windows Service Status and Health

## Purpose
Analyze Windows services including status, startup type, failed services, service crashes, and services that should be running but aren't.

## PowerShell Code

### Complete Service Analysis
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Get all services
        $services = Get-CimInstance -ClassName Win32_Service -ErrorAction Stop
        
        # Categorize services
        $runningServices = $services | Where-Object { $_.State -eq "Running" }
        $stoppedServices = $services | Where-Object { $_.State -eq "Stopped" }
        
        # Services set to Auto but not running (potential issue)
        $shouldBeRunning = $services | Where-Object { 
            $_.StartMode -eq "Auto" -and $_.State -ne "Running"
        }
        
        # Disabled services (for info)
        $disabledServices = $services | Where-Object { $_.StartMode -eq "Disabled" }
        
        # Get service account info
        $privilegedServices = $services | Where-Object { 
            $_.StartName -and 
            $_.StartName -notlike "NT AUTHORITY\*" -and 
            $_.StartName -ne "LocalSystem"
        }
        
        [PSCustomObject]@{
            TotalServices         = $services.Count
            RunningCount          = $runningServices.Count
            StoppedCount          = $stoppedServices.Count
            AutoStartNotRunning   = $shouldBeRunning
            DisabledServices      = $disabledServices
            PrivilegedServices    = $privilegedServices
            AllServices           = $services
            Timestamp             = Get-Date
        }
        
    } catch {
        throw "Service analysis failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nAnalyzing services on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Service Summary ===" -ForegroundColor Yellow
    Write-Host "  Total Services:      $($result.TotalServices)"
    Write-Host "  Running:             $($result.RunningCount)"
    Write-Host "  Stopped:             $($result.StoppedCount)"
    Write-Host "  Disabled:            $($result.DisabledServices.Count)"
    Write-Host ""
    
    if ($result.AutoStartNotRunning) {
        Write-Host "⚠ WARNING: Services set to Auto but NOT running:" -ForegroundColor Red
        $result.AutoStartNotRunning | Format-Table Name, DisplayName, State, Status -AutoSize
        Write-Host "  → These services may have crashed or failed to start" -ForegroundColor Yellow
    } else {
        Write-Host "✓ All Auto-start services are running" -ForegroundColor Green
    }
    
    Write-Host "`n=== Running Services (Top 20) ===" -ForegroundColor Yellow
    $result.AllServices | Where-Object { $_.State -eq "Running" } | 
        Select-Object Name, DisplayName, StartMode, ProcessId -First 20 |
        Format-Table -AutoSize
    
    if ($result.PrivilegedServices) {
        Write-Host "`nServices Running with Custom Accounts:" -ForegroundColor Cyan
        $result.PrivilegedServices | 
            Select-Object Name, DisplayName, StartName, State |
            Format-Table -AutoSize
    }
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to analyze services on $ServerName : $($_.Exception.Message)"
}
```

### Check for Recent Service Crashes
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$DaysBack = 7

$scriptBlock = {
    param($days)
    
    try {
        $startDate = (Get-Date).AddDays(-$days)
        
        # Query System event log for service crashes (Event ID 7031, 7034)
        $serviceCrashes = Get-WinEvent -FilterHashtable @{
            LogName   = 'System'
            Id        = 7031, 7034
            StartTime = $startDate
        } -ErrorAction SilentlyContinue
        
        $crashData = foreach ($event in $serviceCrashes) {
            $serviceName = "Unknown"
            if ($event.Message -match "The (.*?) service") {
                $serviceName = $matches[1]
            }
            
            [PSCustomObject]@{
                TimeCreated = $event.TimeCreated
                EventId     = $event.Id
                ServiceName = $serviceName
                Message     = $event.Message.Split("`n")[0]
            }
        }
        
        $crashData | Sort-Object TimeCreated -Descending
        
    } catch {
        # No crashes or error accessing event log
        @()
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
    
    Write-Host "`nChecking for service crashes in last $DaysBack days..." -ForegroundColor Cyan
    $crashes = Invoke-Command @invokeParams
    
    if ($crashes) {
        Write-Host "`n⚠ SERVICE CRASHES DETECTED:" -ForegroundColor Red
        $crashes | Format-Table TimeCreated, ServiceName, EventId -AutoSize
        Write-Host ""
        
        # Group by service to find repeat offenders
        $crashGroups = $crashes | Group-Object ServiceName | Sort-Object Count -Descending
        Write-Host "Services with multiple crashes:" -ForegroundColor Yellow
        $crashGroups | Where-Object { $_.Count -gt 1 } | 
            Format-Table @{L='Service';E={$_.Name}}, Count -AutoSize
    } else {
        Write-Host "`n✓ No service crashes detected in last $DaysBack days" -ForegroundColor Green
    }
    
    $crashes
    
} catch {
    Write-Error "Failed to check service crashes: $($_.Exception.Message)"
}
```

### Get Service Dependencies
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$ServiceName = "W3SVC"  # Example: IIS service

$scriptBlock = {
    param($svcName)
    
    try {
        $service = Get-Service -Name $svcName -ErrorAction Stop
        
        # Services this service depends on
        $dependencies = $service.ServicesDependedOn
        
        # Services that depend on this service
        $dependents = Get-Service | Where-Object { 
            $_.ServicesDependedOn.Name -contains $svcName 
        }
        
        [PSCustomObject]@{
            ServiceName      = $service.Name
            DisplayName      = $service.DisplayName
            Status           = $service.Status
            StartType        = $service.StartType
            DependsOn        = $dependencies
            DependentServices = $dependents
        }
        
    } catch {
        throw "Failed to get service dependencies: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($ServiceName)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    $svcInfo = Invoke-Command @invokeParams
    
    Write-Host "`n=== Service Dependencies: $ServiceName ===" -ForegroundColor Cyan
    Write-Host "Display Name: $($svcInfo.DisplayName)"
    Write-Host "Status:       $($svcInfo.Status)"
    Write-Host "Start Type:   $($svcInfo.StartType)"
    Write-Host ""
    
    if ($svcInfo.DependsOn) {
        Write-Host "This service depends on:" -ForegroundColor Yellow
        $svcInfo.DependsOn | Format-Table Name, Status, StartType -AutoSize
    } else {
        Write-Host "This service has no dependencies" -ForegroundColor Gray
    }
    
    Write-Host ""
    
    if ($svcInfo.DependentServices) {
        Write-Host "Services that depend on this service:" -ForegroundColor Yellow
        $svcInfo.DependentServices | Format-Table Name, Status, StartType -AutoSize
    } else {
        Write-Host "No services depend on this service" -ForegroundColor Gray
    }
    
    $svcInfo
    
} catch {
    Write-Error "Failed to get service dependencies: $($_.Exception.Message)"
}
```

### Start/Stop/Restart Service (Management)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$ServiceName = "Spooler"  # Example service
$Action = "Restart"  # Start, Stop, Restart

Write-Host "${Action}ing service '$ServiceName' on $ServerName..." -ForegroundColor Cyan

$scriptBlock = {
    param($svc, $action)
    
    try {
        $service = Get-Service -Name $svc -ErrorAction Stop
        $initialState = $service.Status
        
        switch ($action) {
            "Start" {
                if ($service.Status -ne "Running") {
                    Start-Service -Name $svc -ErrorAction Stop
                    Start-Sleep -Seconds 2
                    $service.Refresh()
                }
            }
            "Stop" {
                if ($service.Status -ne "Stopped") {
                    Stop-Service -Name $svc -Force -ErrorAction Stop
                    Start-Sleep -Seconds 2
                    $service.Refresh()
                }
            }
            "Restart" {
                Restart-Service -Name $svc -Force -ErrorAction Stop
                Start-Sleep -Seconds 2
                $service.Refresh()
            }
        }
        
        [PSCustomObject]@{
            ServiceName   = $service.Name
            Action        = $action
            InitialState  = $initialState
            CurrentState  = $service.Status
            Success       = $true
        }
        
    } catch {
        [PSCustomObject]@{
            ServiceName   = $svc
            Action        = $action
            Success       = $false
            Error         = $_.Exception.Message
        }
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($ServiceName, $Action)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    $result = Invoke-Command @invokeParams
    
    if ($result.Success) {
        Write-Host "✓ Service ${Action} successful" -ForegroundColor Green
        Write-Host "  Initial State: $($result.InitialState)"
        Write-Host "  Current State: $($result.CurrentState)"
    } else {
        Write-Warning "✗ Service ${Action} failed: $($result.Error)"
    }
    
    $result
    
} catch {
    Write-Error "Failed to ${Action} service: $($_.Exception.Message)"
}
```

### Find Services by Display Name Pattern
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null
$Pattern = "*SQL*"  # Search pattern

try {
    $result = Invoke-Command -ComputerName $ServerName -Credential $Credential -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        param($pattern)
        
        Get-Service | Where-Object { $_.DisplayName -like $pattern } |
            Select-Object Name, DisplayName, Status, StartType
        
    } -ArgumentList $Pattern -ErrorAction Stop
    
    if ($result) {
        Write-Host "`nServices matching '$Pattern':" -ForegroundColor Cyan
        $result | Format-Table -AutoSize
    } else {
        Write-Host "`nNo services found matching '$Pattern'" -ForegroundColor Yellow
    }
    
    $result
    
} catch {
    Write-Error "Service search failed: $($_.Exception.Message)"
}
```

## Interpreting Results

### Service States

| State | Meaning | Expected |
|-------|---------|----------|
| Running | Service is active | Normal for Auto services |
| Stopped | Service is not running | Normal for Manual/Disabled |
| Paused | Service is paused | Rare; investigate |
| Starting | Service is starting | Transient state |
| Stopping | Service is stopping | Transient state |

### Start Modes

| Mode | Meaning | When Used |
|------|---------|-----------|
| Automatic | Starts at boot | Critical services |
| Automatic (Delayed) | Starts after boot delay | Non-critical services |
| Manual | Starts on demand | On-demand services |
| Disabled | Cannot be started | Unused/security |

### Key Services by Role

**Domain Controller**
- NTDS (Active Directory)
- DNS Server
- Kerberos Key Distribution Center
- Netlogon

**File Server**
- Server (LanmanServer)
- DFS Namespace/Replication

**Web Server (IIS)**
- W3SVC (World Wide Web Publishing)
- WAS (Windows Process Activation)

**SQL Server**
- MSSQLSERVER (or named instance)
- SQLSERVERAGENT

**Print Server**
- Spooler

### Common Issues

| Finding | Likely Problem | Action |
|---------|----------------|--------|
| Auto service stopped | Crash, failure, dependency issue | Check event logs, restart |
| Multiple crashes | Buggy service, configuration error | Investigate logs, update software |
| Service won't start | Dependency issue, permission, corruption | Check dependencies, event logs |
| High process count | Service spawning too many processes | Investigate application |
| Service account issue | Permission denied, password change | Update service credentials |

### Critical Services
These should ALWAYS be running on a domain member:
- **RpcSs** (Remote Procedure Call)
- **Dnscache** (DNS Client)
- **Netlogon** (for domain auth)
- **EventLog** (Event logging)
- **PlugPlay** (Plug and Play)

## Error Handling

```powershell
# Handle services that don't exist or access denied
$ServerName = "TARGET_SERVER"
$ServiceName = "NonExistentService"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        param($svc)
        
        try {
            $service = Get-Service -Name $svc -ErrorAction Stop
            [PSCustomObject]@{
                Found  = $true
                Name   = $service.Name
                Status = $service.Status
            }
        } catch {
            [PSCustomObject]@{
                Found = $false
                Error = $_.Exception.Message
            }
        }
    } -ArgumentList $ServiceName -ErrorAction Stop
    
    if ($result.Found) {
        Write-Host "Service '$ServiceName': $($result.Status)"
    } else {
        Write-Warning "Service '$ServiceName' not found: $($result.Error)"
    }
    
} catch {
    Write-Error "Cannot query services: $($_.Exception.Message)"
}
```

## Next Steps

Based on service analysis:
- **Auto service stopped** → Check **event-logs** for crash reason
- **Service crashes** → Check application logs, update software
- **Performance issues** → Check **processes** for service resource usage
- **Dependency failures** → Check dependent services and **network**
- **Custom account services** → Verify credentials haven't expired
