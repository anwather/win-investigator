# Processes - Running Process Analysis

## Purpose
Analyze running processes on a Windows Server, identifying top CPU and memory consumers, hung/not responding processes, process count, and anomalies.

## PowerShell Code

### Complete Process Analysis
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Get all processes with details
        $processes = Get-CimInstance -ClassName Win32_Process -ErrorAction Stop
        
        # Get process performance data
        $perfProcs = Get-Counter '\Process(*)\% Processor Time' -ErrorAction SilentlyContinue |
            Select-Object -ExpandProperty CounterSamples |
            Where-Object { $_.InstanceName -ne '_total' -and $_.InstanceName -ne 'idle' }
        
        # Build process list with metrics
        $processData = foreach ($proc in $processes) {
            $perfData = $perfProcs | Where-Object { $_.InstanceName -eq $proc.Name }
            
            [PSCustomObject]@{
                ProcessId        = $proc.ProcessId
                Name             = $proc.Name
                CommandLine      = $proc.CommandLine
                WorkingSetMB     = [math]::Round($proc.WorkingSetSize / 1MB, 2)
                VirtualMemoryMB  = [math]::Round($proc.VirtualSize / 1MB, 2)
                ThreadCount      = $proc.ThreadCount
                HandleCount      = $proc.HandleCount
                CPUPercent       = if ($perfData) { [math]::Round($perfData.CookedValue, 2) } else { 0 }
                CreationDate     = $proc.CreationDate
                ParentProcessId  = $proc.ParentProcessId
                ExecutablePath   = $proc.ExecutablePath
            }
        }
        
        # Summary statistics
        $totalProcs = $processData.Count
        $totalWorkingSetGB = [math]::Round(($processData | Measure-Object WorkingSetMB -Sum).Sum / 1024, 2)
        
        # Top consumers
        $topCPU = $processData | Sort-Object CPUPercent -Descending | Select-Object -First 10
        $topMemory = $processData | Sort-Object WorkingSetMB -Descending | Select-Object -First 10
        
        # Potential issues
        $highHandleCount = $processData | Where-Object { $_.HandleCount -gt 10000 }
        $highThreadCount = $processData | Where-Object { $_.ThreadCount -gt 100 }
        
        [PSCustomObject]@{
            TotalProcesses        = $totalProcs
            TotalWorkingSetGB     = $totalWorkingSetGB
            TopCPU                = $topCPU
            TopMemory             = $topMemory
            HighHandleCount       = $highHandleCount
            HighThreadCount       = $highThreadCount
            AllProcesses          = $processData
            Timestamp             = Get-Date
        }
        
    } catch {
        throw "Process analysis failed: $($_.Exception.Message)"
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
    
    if ($Credential) {
        $invokeParams['Credential'] = $Credential
    }
    
    Write-Host "`nAnalyzing processes on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Process Summary ===" -ForegroundColor Yellow
    Write-Host "Total Processes:     $($result.TotalProcesses)"
    Write-Host "Total Memory Usage:  $($result.TotalWorkingSetGB) GB"
    Write-Host ""
    
    Write-Host "=== Top 10 CPU Consumers ===" -ForegroundColor Yellow
    $result.TopCPU | Format-Table ProcessId, Name, @{L='CPU%';E={$_.CPUPercent}}, 
        @{L='MemoryMB';E={$_.WorkingSetMB}}, ThreadCount -AutoSize
    
    Write-Host "=== Top 10 Memory Consumers ===" -ForegroundColor Yellow
    $result.TopMemory | Format-Table ProcessId, Name, @{L='MemoryMB';E={$_.WorkingSetMB}}, 
        @{L='CPU%';E={$_.CPUPercent}}, HandleCount -AutoSize
    
    if ($result.HighHandleCount) {
        Write-Host "`n⚠ Processes with High Handle Count (>10,000):" -ForegroundColor Yellow
        $result.HighHandleCount | Format-Table ProcessId, Name, HandleCount, @{L='MemoryMB';E={$_.WorkingSetMB}} -AutoSize
    }
    
    if ($result.HighThreadCount) {
        Write-Host "`n⚠ Processes with High Thread Count (>100):" -ForegroundColor Yellow
        $result.HighThreadCount | Format-Table ProcessId, Name, ThreadCount, @{L='MemoryMB';E={$_.WorkingSetMB}} -AutoSize
    }
    
    # Return full result for further analysis
    $result
    
} catch {
    Write-Error "Failed to analyze processes on $ServerName : $($_.Exception.Message)"
}
```

### Detect Hung/Not Responding Processes
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    $hungProcesses = @()
    
    # Get all processes with window handles
    $processes = Get-Process | Where-Object { $_.MainWindowHandle -ne 0 }
    
    foreach ($proc in $processes) {
        try {
            # Check if process is responding (Windows GUI apps only)
            if (-not $proc.Responding) {
                $hungProcesses += [PSCustomObject]@{
                    ProcessId     = $proc.Id
                    ProcessName   = $proc.ProcessName
                    MainWindowTitle = $proc.MainWindowTitle
                    StartTime     = $proc.StartTime
                    Responding    = $false
                    MemoryMB      = [math]::Round($proc.WorkingSet64 / 1MB, 2)
                }
            }
        } catch {
            # Process may have exited
        }
    }
    
    $hungProcesses
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
    
    $hungProcs = Invoke-Command @invokeParams
    
    if ($hungProcs) {
        Write-Host "`n⚠ HUNG/NOT RESPONDING PROCESSES DETECTED!" -ForegroundColor Red
        $hungProcs | Format-Table ProcessId, ProcessName, MainWindowTitle, MemoryMB -AutoSize
        Write-Host "These processes may need to be terminated." -ForegroundColor Yellow
    } else {
        Write-Host "`n✓ No hung processes detected" -ForegroundColor Green
    }
    
    $hungProcs
    
} catch {
    Write-Warning "Cannot check for hung processes: $($_.Exception.Message)"
}
```

### Find Processes by Name or Pattern
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null
$ProcessPattern = "sql*"  # e.g., "w3wp", "sql*", "*java*"

try {
    $result = Invoke-Command -ComputerName $ServerName -Credential $Credential -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        param($pattern)
        
        Get-CimInstance Win32_Process | 
            Where-Object { $_.Name -like $pattern } |
            Select-Object ProcessId, Name, 
                @{L='MemoryMB';E={[math]::Round($_.WorkingSetSize/1MB,2)}},
                @{L='VirtualMB';E={[math]::Round($_.VirtualSize/1MB,2)}},
                ThreadCount, HandleCount, CommandLine, CreationDate
        
    } -ArgumentList $ProcessPattern -ErrorAction Stop
    
    if ($result) {
        Write-Host "`nFound $($result.Count) process(es) matching '$ProcessPattern':" -ForegroundColor Cyan
        $result | Format-Table ProcessId, Name, MemoryMB, ThreadCount, HandleCount -AutoSize
    } else {
        Write-Host "`nNo processes found matching '$ProcessPattern'" -ForegroundColor Yellow
    }
    
    $result
    
} catch {
    Write-Error "Failed to search processes: $($_.Exception.Message)"
}
```

### Get Process Tree (Parent-Child Relationships)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    $allProcs = Get-CimInstance Win32_Process
    
    function Get-ProcessTree($ProcessId, $Indent = 0) {
        $proc = $allProcs | Where-Object { $_.ProcessId -eq $ProcessId }
        if ($proc) {
            $output = "  " * $Indent + "$($proc.ProcessId) - $($proc.Name)"
            Write-Output $output
            
            $children = $allProcs | Where-Object { $_.ParentProcessId -eq $ProcessId }
            foreach ($child in $children) {
                Get-ProcessTree -ProcessId $child.ProcessId -Indent ($Indent + 1)
            }
        }
    }
    
    # Find root processes (no parent or parent doesn't exist)
    $rootProcs = $allProcs | Where-Object { 
        $_.ParentProcessId -eq 0 -or 
        -not ($allProcs | Where-Object { $_.ProcessId -eq $_.ParentProcessId })
    }
    
    foreach ($root in $rootProcs | Select-Object -First 5) {
        Get-ProcessTree -ProcessId $root.ProcessId
    }
}

try {
    Invoke-Command -ComputerName $ServerName -Credential $credential -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock $scriptBlock -ErrorAction Stop
} catch {
    Write-Error "Failed to get process tree: $($_.Exception.Message)"
}
```

## Interpreting Results

### Normal Patterns
- **100-300 processes**: Typical Windows Server
- **Top CPU**: Usually system idle or legitimate workload (SQL, IIS, etc.)
- **Top Memory**: Expected for role (SQL Server, Exchange, etc.)

### Warning Signs

| Indicator | Possible Issue |
|-----------|----------------|
| Process count >500 | Potential runaway process spawning |
| Unknown process using >50% CPU | Malware, misconfiguration, or runaway task |
| "w3wp.exe" high CPU/memory | IIS application pool issue or attack |
| "svchost.exe" high resources | Windows service problem |
| High handle count (>10k) | Resource leak |
| High thread count (>100 per process) | Threading issue, potential deadlock |
| Hung processes | Application freeze, needs restart |
| Multiple instances of same process | Normal for some apps, odd for others |

### Common Culprits

**SQL Server (sqlservr.exe)**
- High memory: Expected; SQL uses max server memory setting
- High CPU: Active queries; check query performance

**IIS (w3wp.exe)**
- Multiple instances: One per app pool (normal)
- High CPU: Check for slow web app, bad code, or attack
- Memory leak: Progressive memory growth over time

**System/Idle Process**
- Ignore; it's kernel/idle time measurement

**svchost.exe**
- Multiple instances: Normal (service host for Windows services)
- Use `tasklist /svc` to see which services are in each svchost

**Unknown/Suspicious**
- Processes in Temp folders: Potential malware
- Misspelled system processes (e.g., "svch0st.exe"): Malware
- Random name (e.g., "aXb72.exe"): Investigate

## Error Handling

### Handle Processes That Exit During Enumeration
```powershell
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        Get-CimInstance Win32_Process -ErrorAction SilentlyContinue | ForEach-Object {
            try {
                [PSCustomObject]@{
                    PID  = $_.ProcessId
                    Name = $_.Name
                    MB   = [math]::Round($_.WorkingSetSize/1MB,1)
                }
            } catch {
                # Process exited; skip
            }
        }
    } -ErrorAction Stop
    
    $result | Sort-Object MB -Descending | Select-Object -First 20
    
} catch {
    Write-Error "Process enumeration failed: $($_.Exception.Message)"
}
```

## Next Steps

After process analysis:
- **High CPU usage** → Check **performance** counters for bottlenecks
- **High memory usage** → Check **performance** for available memory
- **Hung processes** → Check **event-logs** for application crashes
- **Unknown processes** → Check **installed-apps** and scan for malware
- **Specific process issues** → Investigate that application's logs
