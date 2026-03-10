---
name: server-overview
description: "Quick system summary including OS, uptime, hardware, and overall health"
---

# Server Overview - Quick System Summary

## Purpose
Get a high-level overview of the target Windows Server including hostname, OS version, uptime, last boot time, domain membership, and basic hardware information.

## PowerShell Code

### Complete System Overview
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Operating System Info
        $os = Get-CimInstance -ClassName Win32_OperatingSystem -ErrorAction Stop
        
        # Computer System Info
        $cs = Get-CimInstance -ClassName Win32_ComputerSystem -ErrorAction Stop
        
        # BIOS Info
        $bios = Get-CimInstance -ClassName Win32_BIOS -ErrorAction Stop
        
        # Calculate uptime
        $bootTime = $os.LastBootUpTime
        $uptime = (Get-Date) - $bootTime
        
        # Build structured result
        [PSCustomObject]@{
            Hostname           = $cs.Name
            FQDN               = "$($cs.Name).$($cs.Domain)"
            Domain             = $cs.Domain
            DomainRole         = switch ($cs.DomainRole) {
                0 { "Standalone Workstation" }
                1 { "Member Workstation" }
                2 { "Standalone Server" }
                3 { "Member Server" }
                4 { "Backup Domain Controller" }
                5 { "Primary Domain Controller" }
                default { "Unknown ($($cs.DomainRole))" }
            }
            PartOfDomain       = $cs.PartOfDomain
            
            OSName             = $os.Caption
            OSVersion          = $os.Version
            OSBuild            = $os.BuildNumber
            OSArchitecture     = $os.OSArchitecture
            ServicePackMajor   = $os.ServicePackMajorVersion
            InstallDate        = $os.InstallDate
            
            LastBootTime       = $bootTime
            UptimeDays         = [math]::Round($uptime.TotalDays, 2)
            UptimeFormatted    = "$($uptime.Days)d $($uptime.Hours)h $($uptime.Minutes)m"
            
            Manufacturer       = $cs.Manufacturer
            Model              = $cs.Model
            TotalPhysicalRAM_GB = [math]::Round($cs.TotalPhysicalMemory / 1GB, 2)
            NumberOfProcessors = $cs.NumberOfProcessors
            NumberOfLogicalProc = $cs.NumberOfLogicalProcessors
            
            BIOSVersion        = $bios.SMBIOSBIOSVersion
            BIOSManufacturer   = $bios.Manufacturer
            
            TimeZone           = $os.CurrentTimeZone / 60  # Hours from UTC
            LocalDateTime      = Get-Date
            
            Status             = "Online"
        }
    } catch {
        [PSCustomObject]@{
            Hostname = $env:COMPUTERNAME
            Status   = "Error"
            Error    = $_.Exception.Message
        }
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
    
    $result = Invoke-Command @invokeParams
    
    # Display results
    Write-Host "`n=== Server Overview: $ServerName ===" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "System Identity:" -ForegroundColor Yellow
    Write-Host "  Hostname:        $($result.Hostname)"
    Write-Host "  FQDN:            $($result.FQDN)"
    Write-Host "  Domain:          $($result.Domain)"
    Write-Host "  Domain Role:     $($result.DomainRole)"
    Write-Host ""
    Write-Host "Operating System:" -ForegroundColor Yellow
    Write-Host "  OS:              $($result.OSName)"
    Write-Host "  Version:         $($result.OSVersion) (Build $($result.OSBuild))"
    Write-Host "  Architecture:    $($result.OSArchitecture)"
    Write-Host "  Installed:       $($result.InstallDate)"
    Write-Host ""
    Write-Host "Uptime:" -ForegroundColor Yellow
    Write-Host "  Last Boot:       $($result.LastBootTime)"
    Write-Host "  Uptime:          $($result.UptimeFormatted) ($($result.UptimeDays) days)"
    
    if ($result.UptimeDays -gt 90) {
        Write-Host "  ⚠ Server has been up for $($result.UptimeDays) days - consider patching/reboot" -ForegroundColor Yellow
    } elseif ($result.UptimeDays -lt 1) {
        Write-Host "  ℹ Recent reboot detected" -ForegroundColor Cyan
    }
    
    Write-Host ""
    Write-Host "Hardware:" -ForegroundColor Yellow
    Write-Host "  Manufacturer:    $($result.Manufacturer)"
    Write-Host "  Model:           $($result.Model)"
    Write-Host "  Total RAM:       $($result.TotalPhysicalRAM_GB) GB"
    Write-Host "  Processors:      $($result.NumberOfProcessors) physical, $($result.NumberOfLogicalProc) logical"
    Write-Host "  BIOS:            $($result.BIOSManufacturer) $($result.BIOSVersion)"
    Write-Host ""
    
    # Return structured object for further processing
    $result
    
} catch {
    Write-Error "Failed to get server overview from $ServerName : $($_.Exception.Message)"
}
```

### Lightweight Version (Fast Check)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

try {
    $result = Invoke-Command -ComputerName $ServerName -Credential $Credential -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        $os = Get-CimInstance Win32_OperatingSystem
        $cs = Get-CimInstance Win32_ComputerSystem
        
        [PSCustomObject]@{
            Name    = $cs.Name
            OS      = $os.Caption
            Build   = $os.BuildNumber
            Boot    = $os.LastBootUpTime
            Uptime  = [math]::Round(((Get-Date) - $os.LastBootUpTime).TotalDays, 1)
            Domain  = $cs.Domain
            RAM_GB  = [math]::Round($cs.TotalPhysicalMemory / 1GB, 1)
        }
    } -ErrorAction Stop
    
    Write-Host "$($result.Name): $($result.OS) | Up $($result.Uptime)d | $($result.RAM_GB) GB RAM"
    $result
    
} catch {
    Write-Warning "Cannot reach $ServerName : $($_.Exception.Message)"
}
```

## Interpreting Results

### Key Health Indicators

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Uptime | 7-60 days | 60-90 days | >90 days or <1 day (frequent reboots) |
| Domain Role | Member Server/DC | Standalone | Wrong role type |
| Install Date | Recent patches | >6 months old | >1 year old |

### What to Look For

1. **Uptime Concerns**
   - **>90 days**: Likely missing Windows Updates that require reboot
   - **<1 day with issues**: Recent reboot might indicate crash or problem
   - **Very long uptime**: Security risk; patches not applied

2. **Domain Membership**
   - **Not in domain when expected**: Group Policy not applying, authentication issues
   - **Wrong domain**: Configuration error
   - **Domain role mismatch**: Server configured incorrectly

3. **OS Version/Build**
   - Check against Microsoft lifecycle: https://aka.ms/windowsserver-lifecycle
   - Out-of-support versions are security risks
   - Old builds missing critical patches

4. **Hardware**
   - Low RAM (<4GB): Likely performance issues
   - Manufacturer "VMware" or "Microsoft Corporation": Virtual machine
   - Physical vs Virtual matters for troubleshooting

### Common Issues Revealed

| Finding | Likely Problem | Action |
|---------|---------------|--------|
| Uptime >100 days | Missing patches, updates not applied | Schedule maintenance window |
| PartOfDomain = False | Not domain joined | Check network, rejoin domain |
| Recent boot (<1 hour) with errors | System crashed and auto-rebooted | Check event logs for crash dump |
| Old OS build | Unpatched system | Run Windows Update |
| Low RAM with high page file use | Memory pressure | Add RAM or reduce workload |

## Error Handling

```powershell
# Timeout after 30 seconds if server is slow/hung
$ServerName = "TARGET_SERVER"

$job = Start-Job -ScriptBlock {
    param($srv)
    Invoke-Command -ComputerName $srv -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, LastBootUpTime
    }
} -ArgumentList $ServerName

if (Wait-Job -Job $job -Timeout 30) {
    $result = Receive-Job -Job $job
    $result
} else {
    Write-Warning "Connection to $ServerName timed out after 30 seconds"
    Stop-Job -Job $job
}

Remove-Job -Job $job -Force
```

## Next Steps

Based on overview results, proceed to:
- **High uptime** → Check **services** and **event-logs** for deferred issues
- **Recent reboot** → Check **event-logs** for crash/unexpected shutdown
- **Domain issues** → Check **network** connectivity and DNS
- **Low RAM** → Check **processes** and **performance** for memory usage
- **Physical server** → Check **disk-storage** for hardware health (SMART)
