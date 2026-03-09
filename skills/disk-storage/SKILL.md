# Disk Storage - Disk Space and Health Analysis

## Purpose
Analyze disk storage including volume space, disk health, SMART data, and identify potential storage issues.

## PowerShell Code

### Complete Disk Storage Analysis
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null  # Set if needed

$scriptBlock = {
    try {
        # Get volume information
        $volumes = Get-Volume | Where-Object { $_.DriveLetter -or $_.FileSystemLabel }
        
        $volumeData = foreach ($vol in $volumes) {
            $driveLetter = if ($vol.DriveLetter) { "$($vol.DriveLetter):" } else { "N/A" }
            
            [PSCustomObject]@{
                DriveLetter     = $driveLetter
                FileSystemLabel = $vol.FileSystemLabel
                FileSystem      = $vol.FileSystem
                SizeGB          = [math]::Round($vol.Size / 1GB, 2)
                FreeGB          = [math]::Round($vol.SizeRemaining / 1GB, 2)
                UsedGB          = [math]::Round(($vol.Size - $vol.SizeRemaining) / 1GB, 2)
                PercentFree     = [math]::Round(($vol.SizeRemaining / $vol.Size) * 100, 2)
                HealthStatus    = $vol.HealthStatus
                OperationalStatus = $vol.OperationalStatus
            }
        }
        
        # Get physical disk information
        $physicalDisks = Get-PhysicalDisk -ErrorAction SilentlyContinue
        
        $diskData = foreach ($disk in $physicalDisks) {
            [PSCustomObject]@{
                FriendlyName      = $disk.FriendlyName
                MediaType         = $disk.MediaType
                BusType           = $disk.BusType
                SizeGB            = [math]::Round($disk.Size / 1GB, 2)
                HealthStatus      = $disk.HealthStatus
                OperationalStatus = $disk.OperationalStatus
                Usage             = $disk.Usage
                SerialNumber      = $disk.SerialNumber
            }
        }
        
        # Get partition information
        $partitions = Get-Partition -ErrorAction SilentlyContinue | Where-Object { $_.DriveLetter }
        
        $partitionData = foreach ($part in $partitions) {
            [PSCustomObject]@{
                DriveLetter   = "$($part.DriveLetter):"
                PartitionNumber = $part.PartitionNumber
                SizeGB        = [math]::Round($part.Size / 1GB, 2)
                Type          = $part.Type
                IsSystem      = $part.IsSystem
                IsActive      = $part.IsActive
                IsBoot        = $part.IsBoot
            }
        }
        
        # Check for low disk space
        $lowSpace = $volumeData | Where-Object { $_.PercentFree -lt 15 -and $_.SizeGB -gt 1 }
        $criticalSpace = $volumeData | Where-Object { $_.PercentFree -lt 5 -and $_.SizeGB -gt 1 }
        
        # Check for unhealthy disks
        $unhealthyDisks = $diskData | Where-Object { $_.HealthStatus -ne "Healthy" }
        
        [PSCustomObject]@{
            Volumes          = $volumeData
            PhysicalDisks    = $diskData
            Partitions       = $partitionData
            LowSpaceVolumes  = $lowSpace
            CriticalSpaceVolumes = $criticalSpace
            UnhealthyDisks   = $unhealthyDisks
            Timestamp        = Get-Date
        }
        
    } catch {
        throw "Disk storage analysis failed: $($_.Exception.Message)"
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
    
    Write-Host "`nAnalyzing disk storage on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Volume Information ===" -ForegroundColor Yellow
    $result.Volumes | Format-Table DriveLetter, FileSystemLabel, SizeGB, FreeGB, PercentFree, HealthStatus -AutoSize
    
    Write-Host "`n=== Physical Disks ===" -ForegroundColor Yellow
    $result.PhysicalDisks | Format-Table FriendlyName, MediaType, BusType, SizeGB, HealthStatus, OperationalStatus -AutoSize
    
    if ($result.CriticalSpaceVolumes) {
        Write-Host "`n⚠ CRITICAL: Volumes with <5% free space!" -ForegroundColor Red
        $result.CriticalSpaceVolumes | Format-Table DriveLetter, FileSystemLabel, FreeGB, PercentFree -AutoSize
    }
    
    if ($result.LowSpaceVolumes) {
        Write-Host "`n⚠ WARNING: Volumes with <15% free space" -ForegroundColor Yellow
        $result.LowSpaceVolumes | Format-Table DriveLetter, FileSystemLabel, FreeGB, PercentFree -AutoSize
    }
    
    if ($result.UnhealthyDisks) {
        Write-Host "`n⚠ UNHEALTHY DISKS DETECTED!" -ForegroundColor Red
        $result.UnhealthyDisks | Format-Table FriendlyName, HealthStatus, OperationalStatus -AutoSize
        Write-Host "  → Immediate action required! Backup data and replace disk." -ForegroundColor Yellow
    }
    
    if (-not $result.CriticalSpaceVolumes -and -not $result.LowSpaceVolumes -and -not $result.UnhealthyDisks) {
        Write-Host "`n✓ All disks healthy with adequate free space" -ForegroundColor Green
    }
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to analyze disk storage on $ServerName : $($_.Exception.Message)"
}
```

### Get SMART Data (Physical Disk Health)
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null

$scriptBlock = {
    try {
        # Get SMART data via Storage cmdlets
        $disks = Get-PhysicalDisk
        
        $smartData = foreach ($disk in $disks) {
            $reliability = Get-StorageReliabilityCounter -PhysicalDisk $disk -ErrorAction SilentlyContinue
            
            [PSCustomObject]@{
                DiskNumber        = $disk.DeviceId
                FriendlyName      = $disk.FriendlyName
                MediaType         = $disk.MediaType
                HealthStatus      = $disk.HealthStatus
                Temperature       = $reliability.Temperature
                Wear              = $reliability.Wear
                ReadErrors        = $reliability.ReadErrorsTotal
                WriteErrors       = $reliability.WriteErrorsTotal
                PowerOnHours      = $reliability.PowerOnHours
                StartStopCycles   = $reliability.StartStopCycleCount
            }
        }
        
        $smartData
        
    } catch {
        throw "SMART data collection failed: $($_.Exception.Message)"
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
    
    Write-Host "`nRetrieving SMART data from $ServerName..." -ForegroundColor Cyan
    $smartData = Invoke-Command @invokeParams
    
    if ($smartData) {
        Write-Host "`n=== SMART Data / Disk Health ===" -ForegroundColor Yellow
        $smartData | Format-Table FriendlyName, MediaType, HealthStatus, Temperature, 
            PowerOnHours, ReadErrors, WriteErrors -AutoSize
        
        # Check for issues
        $hotDisks = $smartData | Where-Object { $_.Temperature -gt 60 }
        $errorDisks = $smartData | Where-Object { $_.ReadErrors -gt 0 -or $_.WriteErrors -gt 0 }
        
        if ($hotDisks) {
            Write-Host "`n⚠ Disks running hot (>60°C):" -ForegroundColor Yellow
            $hotDisks | Format-Table FriendlyName, Temperature -AutoSize
        }
        
        if ($errorDisks) {
            Write-Host "`n⚠ Disks with read/write errors:" -ForegroundColor Red
            $errorDisks | Format-Table FriendlyName, ReadErrors, WriteErrors -AutoSize
            Write-Host "  → These disks may be failing. Backup and replace immediately." -ForegroundColor Yellow
        }
    } else {
        Write-Host "`nSMART data not available (may not be supported on this system)" -ForegroundColor Yellow
    }
    
    $smartData
    
} catch {
    Write-Warning "SMART data retrieval failed: $($_.Exception.Message)"
}
```

### Find Large Files/Folders
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null
$DriveLetter = "C"
$TopN = 20

Write-Host "Finding largest items on ${DriveLetter}: drive..." -ForegroundColor Cyan
Write-Host "Note: This can take several minutes on large volumes" -ForegroundColor Yellow

$scriptBlock = {
    param($drive, $top)
    
    try {
        # Find largest folders
        $folders = Get-ChildItem -Path "${drive}:\" -Directory -Force -ErrorAction SilentlyContinue |
            ForEach-Object {
                $size = (Get-ChildItem -Path $_.FullName -Recurse -File -Force -ErrorAction SilentlyContinue |
                    Measure-Object -Property Length -Sum -ErrorAction SilentlyContinue).Sum
                
                [PSCustomObject]@{
                    Type   = "Folder"
                    Path   = $_.FullName
                    SizeGB = [math]::Round($size / 1GB, 2)
                }
            } | Sort-Object SizeGB -Descending | Select-Object -First $top
        
        # Find largest files
        $files = Get-ChildItem -Path "${drive}:\" -File -Recurse -Force -ErrorAction SilentlyContinue |
            Sort-Object Length -Descending |
            Select-Object -First $top |
            ForEach-Object {
                [PSCustomObject]@{
                    Type   = "File"
                    Path   = $_.FullName
                    SizeGB = [math]::Round($_.Length / 1GB, 2)
                }
            }
        
        # Combine and return top items
        ($folders + $files) | Sort-Object SizeGB -Descending | Select-Object -First $top
        
    } catch {
        throw "Large file search failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($DriveLetter, $TopN)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($Credential) {
        $invokeParams['Credential'] = $Credential
    }
    
    $largeItems = Invoke-Command @invokeParams
    
    Write-Host "`n=== Top $TopN Largest Items on ${DriveLetter}: ===" -ForegroundColor Yellow
    $largeItems | Format-Table Type, SizeGB, Path -AutoSize
    
    $largeItems
    
} catch {
    Write-Error "Failed to find large files: $($_.Exception.Message)"
}
```

### Check Disk I/O Statistics
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null

$scriptBlock = {
    try {
        $logicalDisks = Get-CimInstance Win32_LogicalDisk | Where-Object { $_.DriveType -eq 3 }
        
        foreach ($disk in $logicalDisks) {
            $perfData = Get-Counter "\LogicalDisk($($disk.DeviceID))\*" -ErrorAction SilentlyContinue
            
            if ($perfData) {
                $samples = $perfData.CounterSamples
                
                [PSCustomObject]@{
                    Drive             = $disk.DeviceID
                    ReadsPerSec       = [math]::Round(($samples | Where-Object { $_.Path -like "*Disk Reads/sec" }).CookedValue, 2)
                    WritesPerSec      = [math]::Round(($samples | Where-Object { $_.Path -like "*Disk Writes/sec" }).CookedValue, 2)
                    AvgDiskSecRead    = [math]::Round(($samples | Where-Object { $_.Path -like "*Avg. Disk sec/Read" }).CookedValue * 1000, 2)
                    AvgDiskSecWrite   = [math]::Round(($samples | Where-Object { $_.Path -like "*Avg. Disk sec/Write" }).CookedValue * 1000, 2)
                    DiskQueueLength   = [math]::Round(($samples | Where-Object { $_.Path -like "*Avg. Disk Queue Length" }).CookedValue, 2)
                }
            }
        }
        
    } catch {
        throw "Disk I/O statistics failed: $($_.Exception.Message)"
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
    
    Write-Host "`nDisk I/O Performance:" -ForegroundColor Cyan
    $ioStats = Invoke-Command @invokeParams
    
    $ioStats | Format-Table Drive, ReadsPerSec, WritesPerSec, 
        @{L='ReadLatency(ms)';E={$_.AvgDiskSecRead}}, 
        @{L='WriteLatency(ms)';E={$_.AvgDiskSecWrite}}, 
        DiskQueueLength -AutoSize
    
    $slowDisks = $ioStats | Where-Object { $_.AvgDiskSecRead -gt 20 -or $_.AvgDiskSecWrite -gt 20 }
    if ($slowDisks) {
        Write-Host "`n⚠ Slow disks detected (latency >20ms):" -ForegroundColor Yellow
        $slowDisks | Format-Table Drive, @{L='ReadLatency(ms)';E={$_.AvgDiskSecRead}}, 
            @{L='WriteLatency(ms)';E={$_.AvgDiskSecWrite}} -AutoSize
    }
    
    $ioStats
    
} catch {
    Write-Error "Failed to get disk I/O statistics: $($_.Exception.Message)"
}
```

## Interpreting Results

### Disk Space Thresholds

| Free Space | Status | Action Required |
|------------|--------|-----------------|
| >20% | Healthy | Monitor regularly |
| 10-20% | Warning | Plan cleanup or expansion |
| 5-10% | Critical | Immediate action needed |
| <5% | Emergency | System may become unstable |

### Special Volume Considerations
- **System (C:)**: Keep >15% free for updates, temp files, crash dumps
- **Database volumes**: Keep >20% free for growth, index rebuilds
- **Log volumes**: Monitor actively; logs can fill rapidly

### Health Status Meanings

| Status | Meaning | Action |
|--------|---------|--------|
| Healthy | Normal operation | None |
| Warning | Predictive failure detected | Backup and monitor closely |
| Unhealthy | Disk failing | Replace immediately |
| Unknown | Cannot determine health | Investigate |

### SMART Indicators

**Temperature**
- <50°C: Normal
- 50-60°C: Acceptable
- >60°C: Hot (check cooling, may shorten lifespan)

**Read/Write Errors**
- 0: Good
- >0: Disk problems; potential failure

**Power-On Hours**
- Consumer drives: ~40,000 hours (~4.5 years) typical lifespan
- Enterprise drives: ~100,000+ hours

### I/O Performance

| Metric | SSD Good | HDD Good | Problem |
|--------|----------|----------|---------|
| Read Latency | <10ms | <15ms | >50ms |
| Write Latency | <10ms | <15ms | >50ms |
| Queue Length | <1 | <2 | >5 |

## Common Issues

### Disk Full
**Causes**: Log files, temp files, backups, page file, hibernation file
**Quick wins**: Clean temp, old logs, Windows Update cleanup, disk cleanup utility

### Disk Fragmentation (HDD only)
**Symptoms**: Slow performance, high latency
**Action**: Defragmentation (not needed on SSDs)

### Failing Disk
**Symptoms**: Unhealthy status, SMART errors, slow performance, clicking sounds
**Action**: Backup immediately, replace disk

### Storage Configuration Issues
**Symptoms**: Single large volume, no redundancy
**Action**: Consider RAID, multiple volumes for better management

## Error Handling

```powershell
# Gracefully handle missing cmdlets or permissions
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        try {
            Get-Volume | Select-Object DriveLetter, 
                @{L='SizeGB';E={[math]::Round($_.Size/1GB,2)}}, 
                @{L='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,2)}}, 
                HealthStatus
        } catch {
            # Fallback to WMI if Storage cmdlets unavailable
            Get-CimInstance Win32_LogicalDisk | Where-Object { $_.DriveType -eq 3 } |
                Select-Object DeviceID, 
                    @{L='SizeGB';E={[math]::Round($_.Size/1GB,2)}}, 
                    @{L='FreeGB';E={[math]::Round($_.FreeSpace/1GB,2)}}
        }
    } -ErrorAction Stop
    
    $result | Format-Table -AutoSize
    
} catch {
    Write-Error "Disk check failed: $($_.Exception.Message)"
}
```

## Next Steps

Based on disk analysis:
- **Low space** → Find large files, clean up, or expand volume
- **Unhealthy disk** → Immediate backup and replacement
- **High I/O latency** → Check **performance** counters for bottleneck confirmation
- **SMART errors** → Replace disk before failure
- **Log growth** → Check **services** and applications generating logs
