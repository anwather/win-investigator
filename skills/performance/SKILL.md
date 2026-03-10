# Performance - System Performance Metrics

## Purpose
Collect real-time and historical performance metrics including CPU usage, memory usage, disk I/O, network throughput, and performance counters.

## PowerShell Code

### Comprehensive Performance Snapshot
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # CPU Performance
        $cpuCounter = Get-Counter '\Processor(_Total)\% Processor Time' -ErrorAction Stop
        $cpuAvg = [math]::Round($cpuCounter.CounterSamples[0].CookedValue, 2)
        
        $cpuQueue = Get-Counter '\System\Processor Queue Length' -ErrorAction SilentlyContinue
        $queueLength = if ($cpuQueue) { $cpuQueue.CounterSamples[0].CookedValue } else { 0 }
        
        # Memory Performance
        $os = Get-CimInstance Win32_OperatingSystem
        $totalMemGB = [math]::Round($os.TotalVisibleMemorySize / 1MB, 2)
        $freeMemGB = [math]::Round($os.FreePhysicalMemory / 1MB, 2)
        $usedMemGB = $totalMemGB - $freeMemGB
        $memPercent = [math]::Round(($usedMemGB / $totalMemGB) * 100, 2)
        
        $availableMemMB = Get-Counter '\Memory\Available MBytes' -ErrorAction SilentlyContinue
        $availableMB = if ($availableMemMB) { $availableMemMB.CounterSamples[0].CookedValue } else { $freeMemGB * 1024 }
        
        $pageFaults = Get-Counter '\Memory\Page Faults/sec' -ErrorAction SilentlyContinue
        $pagesSec = Get-Counter '\Memory\Pages/sec' -ErrorAction SilentlyContinue
        
        # Disk Performance
        $diskReads = Get-Counter '\PhysicalDisk(_Total)\Disk Reads/sec' -ErrorAction SilentlyContinue
        $diskWrites = Get-Counter '\PhysicalDisk(_Total)\Disk Writes/sec' -ErrorAction SilentlyContinue
        $diskQueue = Get-Counter '\PhysicalDisk(_Total)\Avg. Disk Queue Length' -ErrorAction SilentlyContinue
        $diskLatency = Get-Counter '\PhysicalDisk(_Total)\Avg. Disk sec/Transfer' -ErrorAction SilentlyContinue
        
        # Network Performance
        $netBytesSent = Get-Counter '\Network Interface(*)\Bytes Sent/sec' -ErrorAction SilentlyContinue
        $netBytesRecv = Get-Counter '\Network Interface(*)\Bytes Received/sec' -ErrorAction SilentlyContinue
        
        $totalSent = ($netBytesSent.CounterSamples | Where-Object { $_.InstanceName -notlike '*isatap*' -and $_.InstanceName -notlike '*Pseudo*' } | 
            Measure-Object CookedValue -Sum).Sum
        $totalRecv = ($netBytesRecv.CounterSamples | Where-Object { $_.InstanceName -notlike '*isatap*' -and $_.InstanceName -notlike '*Pseudo*' } | 
            Measure-Object CookedValue -Sum).Sum
        
        [PSCustomObject]@{
            # CPU
            CPU_PercentUsed         = $cpuAvg
            CPU_QueueLength         = $queueLength
            CPU_Status              = if ($cpuAvg -gt 90) { "Critical" } elseif ($cpuAvg -gt 70) { "Warning" } else { "Normal" }
            
            # Memory
            Memory_TotalGB          = $totalMemGB
            Memory_UsedGB           = $usedMemGB
            Memory_FreeGB           = $freeMemGB
            Memory_PercentUsed      = $memPercent
            Memory_AvailableMB      = $availableMB
            Memory_PageFaultsPerSec = if ($pageFaults) { [math]::Round($pageFaults.CounterSamples[0].CookedValue, 2) } else { 0 }
            Memory_PagesPerSec      = if ($pagesSec) { [math]::Round($pagesSec.CounterSamples[0].CookedValue, 2) } else { 0 }
            Memory_Status           = if ($memPercent -gt 90) { "Critical" } elseif ($memPercent -gt 80) { "Warning" } else { "Normal" }
            
            # Disk
            Disk_ReadsPerSec        = if ($diskReads) { [math]::Round($diskReads.CounterSamples[0].CookedValue, 2) } else { 0 }
            Disk_WritesPerSec       = if ($diskWrites) { [math]::Round($diskWrites.CounterSamples[0].CookedValue, 2) } else { 0 }
            Disk_QueueLength        = if ($diskQueue) { [math]::Round($diskQueue.CounterSamples[0].CookedValue, 2) } else { 0 }
            Disk_LatencyMs          = if ($diskLatency) { [math]::Round($diskLatency.CounterSamples[0].CookedValue * 1000, 2) } else { 0 }
            Disk_Status             = if ($diskLatency -and $diskLatency.CounterSamples[0].CookedValue -gt 0.050) { "Slow" } else { "Normal" }
            
            # Network
            Network_BytesSentPerSec = [math]::Round($totalSent, 2)
            Network_BytesRecvPerSec = [math]::Round($totalRecv, 2)
            Network_MbpsSent        = [math]::Round($totalSent * 8 / 1MB, 2)
            Network_MbpsRecv        = [math]::Round($totalRecv * 8 / 1MB, 2)
            
            Timestamp               = Get-Date
        }
        
    } catch {
        throw "Performance monitoring failed: $($_.Exception.Message)"
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
    
    Write-Host "`nGathering performance metrics from $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== CPU Performance ===" -ForegroundColor Yellow
    Write-Host "  Utilization:     $($result.CPU_PercentUsed)% [$($result.CPU_Status)]"
    Write-Host "  Queue Length:    $($result.CPU_QueueLength)"
    
    if ($result.CPU_PercentUsed -gt 80) {
        Write-Host "  ⚠ High CPU usage detected" -ForegroundColor Yellow
    }
    
    Write-Host "`n=== Memory Performance ===" -ForegroundColor Yellow
    Write-Host "  Total:           $($result.Memory_TotalGB) GB"
    Write-Host "  Used:            $($result.Memory_UsedGB) GB ($($result.Memory_PercentUsed)%)"
    Write-Host "  Available:       $($result.Memory_FreeGB) GB [$($result.Memory_Status)]"
    Write-Host "  Page Faults/sec: $($result.Memory_PageFaultsPerSec)"
    Write-Host "  Pages/sec:       $($result.Memory_PagesPerSec)"
    
    if ($result.Memory_PagesPerSec -gt 100) {
        Write-Host "  ⚠ High paging activity - potential memory pressure" -ForegroundColor Yellow
    }
    
    Write-Host "`n=== Disk Performance ===" -ForegroundColor Yellow
    Write-Host "  Reads/sec:       $($result.Disk_ReadsPerSec)"
    Write-Host "  Writes/sec:      $($result.Disk_WritesPerSec)"
    Write-Host "  Queue Length:    $($result.Disk_QueueLength)"
    Write-Host "  Latency:         $($result.Disk_LatencyMs) ms [$($result.Disk_Status)]"
    
    if ($result.Disk_LatencyMs -gt 50) {
        Write-Host "  ⚠ High disk latency detected - storage bottleneck" -ForegroundColor Red
    }
    
    Write-Host "`n=== Network Performance ===" -ForegroundColor Yellow
    Write-Host "  Sending:         $($result.Network_MbpsSent) Mbps"
    Write-Host "  Receiving:       $($result.Network_MbpsRecv) Mbps"
    Write-Host ""
    
    # Return result for further processing
    $result
    
} catch {
    Write-Error "Failed to gather performance metrics from $ServerName : $($_.Exception.Message)"
}
```

### Continuous Monitoring (Sample Over Time)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$SampleCount = 5
$IntervalSeconds = 2

Write-Host "Collecting $SampleCount samples at $IntervalSeconds second intervals..." -ForegroundColor Cyan

$scriptBlock = {
    param($count, $interval)
    
    $samples = @()
    
    for ($i = 1; $i -le $count; $i++) {
        $cpu = (Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples[0].CookedValue
        $mem = Get-CimInstance Win32_OperatingSystem
        $memPercent = [math]::Round((($mem.TotalVisibleMemorySize - $mem.FreePhysicalMemory) / $mem.TotalVisibleMemorySize) * 100, 2)
        
        $samples += [PSCustomObject]@{
            Sample      = $i
            Time        = Get-Date -Format "HH:mm:ss"
            CPU_Percent = [math]::Round($cpu, 2)
            Mem_Percent = $memPercent
        }
        
        if ($i -lt $count) {
            Start-Sleep -Seconds $interval
        }
    }
    
    $samples
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($SampleCount, $IntervalSeconds)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    $samples = Invoke-Command @invokeParams
    
    Write-Host "`nPerformance Samples:" -ForegroundColor Yellow
    $samples | Format-Table Sample, Time, CPU_Percent, Mem_Percent -AutoSize
    
    $avgCPU = [math]::Round(($samples | Measure-Object CPU_Percent -Average).Average, 2)
    $avgMem = [math]::Round(($samples | Measure-Object Mem_Percent -Average).Average, 2)
    
    Write-Host "`nAverages over $SampleCount samples:" -ForegroundColor Cyan
    Write-Host "  CPU: $avgCPU%"
    Write-Host "  Memory: $avgMem%"
    
    $samples
    
} catch {
    Write-Error "Continuous monitoring failed: $($_.Exception.Message)"
}
```

### Detailed Counter Collection
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

# Define counters of interest
$counters = @(
    '\Processor(_Total)\% Processor Time'
    '\Processor(_Total)\% User Time'
    '\Processor(_Total)\% Interrupt Time'
    '\System\Processor Queue Length'
    '\System\Context Switches/sec'
    '\Memory\Available MBytes'
    '\Memory\Pages/sec'
    '\Memory\Page Faults/sec'
    '\Memory\Cache Bytes'
    '\PhysicalDisk(_Total)\Avg. Disk sec/Read'
    '\PhysicalDisk(_Total)\Avg. Disk sec/Write'
    '\PhysicalDisk(_Total)\Disk Reads/sec'
    '\PhysicalDisk(_Total)\Disk Writes/sec'
)

try {
    $result = Invoke-Command -ComputerName $ServerName -Credential $Credential -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        param($counterList)
        
        $counterData = Get-Counter -Counter $counterList -ErrorAction SilentlyContinue
        
        $counterData.CounterSamples | ForEach-Object {
            [PSCustomObject]@{
                Counter   = $_.Path.Split('\')[-1]
                Category  = $_.Path.Split('\')[-2]
                Value     = [math]::Round($_.CookedValue, 4)
                Timestamp = $_.Timestamp
            }
        }
    } -ArgumentList (,$counters) -ErrorAction Stop
    
    Write-Host "`n=== Detailed Performance Counters ===" -ForegroundColor Cyan
    $result | Format-Table Category, Counter, Value -GroupBy Category -AutoSize
    
    $result
    
} catch {
    Write-Error "Failed to collect performance counters: $($_.Exception.Message)"
}
```

## Interpreting Results

### CPU Metrics

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| % Processor Time | <70% | 70-90% | >90% |
| Processor Queue | <2 per core | 2-5 per core | >5 per core |
| Context Switches/sec | Varies | Sudden spikes | Constant high |

**High CPU causes**: Runaway process, sustained load, malware, inefficient code

### Memory Metrics

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Available Memory | >20% total | 10-20% | <10% |
| Pages/sec | <100 | 100-500 | >500 |
| Page Faults/sec | <1000 | Varies | With high Pages/sec |

**Pages/sec >100**: Memory pressure; system swapping to disk
**Available <512MB**: Critical; imminent out-of-memory condition

### Disk Metrics

| Metric | SSD Healthy | HDD Healthy | Slow |
|--------|-------------|-------------|------|
| Avg Disk sec/Transfer | <10ms | <20ms | >50ms |
| Queue Length | <2 | <2 | >5 |

**High latency + high queue**: Storage bottleneck (slow disks, overloaded SAN, failing drive)

### Network Metrics

- Monitor against link capacity (1 Gbps = 125 MBps, 10 Gbps = 1250 MBps)
- Sudden drop in throughput: Network issue
- Sustained high utilization: Expected load or potential DDoS

## Common Bottlenecks

### CPU-Bound
- Symptoms: High CPU%, low disk/network, process using CPU
- Causes: Inefficient code, intensive computation, malware
- Action: Identify process, optimize or scale

### Memory-Bound
- Symptoms: High memory%, high Pages/sec, available memory low
- Causes: Memory leak, insufficient RAM, too many processes
- Action: Add RAM, fix leak, reduce load

### Disk-Bound
- Symptoms: High disk latency, high queue, CPU waiting
- Causes: Slow storage, disk failure, excessive I/O
- Action: Upgrade storage, check disk health, optimize queries

### Network-Bound
- Symptoms: Low CPU/disk, network saturated
- Causes: Data transfer, replication, network configuration
- Action: Upgrade bandwidth, optimize traffic

## Error Handling

```powershell
# Handle missing performance counters gracefully
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        $counters = @{
            CPU = '\Processor(_Total)\% Processor Time'
            Mem = '\Memory\Available MBytes'
        }
        
        foreach ($key in $counters.Keys) {
            try {
                $value = (Get-Counter $counters[$key] -ErrorAction Stop).CounterSamples[0].CookedValue
                Write-Output "$key = $([math]::Round($value, 2))"
            } catch {
                Write-Output "$key = ERROR"
            }
        }
    } -ErrorAction Stop
    
    $result
    
} catch {
    Write-Error "Performance check failed: $($_.Exception.Message)"
}
```

## Next Steps

Based on performance data:
- **High CPU** → Check **processes** for top CPU consumers
- **High memory** → Check **processes** for memory leaks
- **High disk latency** → Check **disk-storage** for health issues
- **Memory pressure** → Check **services** for unnecessary services
- **Sustained issues** → Check **event-logs** for related errors
