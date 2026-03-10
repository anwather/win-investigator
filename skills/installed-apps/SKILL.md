---
name: installed-apps
description: "Inventory installed applications and software on Windows Servers"
---

# Installed Applications - Software Inventory

## Purpose
Get a list of installed applications, recent installations/updates, software versions, and identify potentially problematic software.

**Performance:** SLOW (5-10s with registry method, 30-120s with Win32_Product)

⚠️ **IMPORTANT:** This skill uses the fast registry-based method (5-10s). NEVER use `Get-CimInstance Win32_Product` as it triggers MSI consistency checks and takes 30-120 seconds. Only run when user specifically asks about installed software.

## PowerShell Code

### Complete Installed Applications Inventory
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Method 1: Registry-based (most reliable for traditional apps)
        $registryPaths = @(
            'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
            'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
        )
        
        $installedApps = foreach ($path in $registryPaths) {
            Get-ItemProperty -Path $path -ErrorAction SilentlyContinue |
                Where-Object { $_.DisplayName } |
                Select-Object DisplayName, DisplayVersion, Publisher, InstallDate, 
                    @{L='InstallLocation';E={$_.InstallLocation}},
                    @{L='UninstallString';E={$_.UninstallString}},
                    @{L='EstimatedSizeMB';E={[math]::Round($_.EstimatedSize/1KB,2)}}
        }
        
        # Remove duplicates and sort
        $uniqueApps = $installedApps | 
            Sort-Object DisplayName -Unique | 
            Sort-Object DisplayName
        
        # Parse install dates (format: YYYYMMDD)
        $appsWithDates = $uniqueApps | ForEach-Object {
            $installDateParsed = $null
            if ($_.InstallDate) {
                try {
                    $year = $_.InstallDate.Substring(0,4)
                    $month = $_.InstallDate.Substring(4,2)
                    $day = $_.InstallDate.Substring(6,2)
                    $installDateParsed = Get-Date "$year-$month-$day" -ErrorAction SilentlyContinue
                } catch {
                    # Invalid date format
                }
            }
            
            $_ | Add-Member -NotePropertyName 'InstallDateParsed' -NotePropertyValue $installDateParsed -PassThru
        }
        
        # Recent installations (last 30 days)
        $recentApps = $appsWithDates | Where-Object { 
            $_.InstallDateParsed -and $_.InstallDateParsed -gt (Get-Date).AddDays(-30) 
        }
        
        [PSCustomObject]@{
            TotalApplications = $uniqueApps.Count
            AllApplications   = $appsWithDates
            RecentInstalls    = $recentApps
            Timestamp         = Get-Date
        }
        
    } catch {
        throw "Application inventory failed: $($_.Exception.Message)"
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
    
    Write-Host "`nInventorying installed applications on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Installed Applications Summary ===" -ForegroundColor Yellow
    Write-Host "  Total Applications: $($result.TotalApplications)"
    Write-Host ""
    
    if ($result.RecentInstalls) {
        Write-Host "=== Recently Installed (Last 30 Days) ===" -ForegroundColor Yellow
        $result.RecentInstalls | 
            Select-Object DisplayName, DisplayVersion, Publisher, InstallDateParsed |
            Format-Table -AutoSize
    } else {
        Write-Host "No recent installations detected (last 30 days)" -ForegroundColor Gray
    }
    
    Write-Host "`n=== All Installed Applications (Top 50) ===" -ForegroundColor Yellow
    $result.AllApplications | 
        Select-Object DisplayName, DisplayVersion, Publisher -First 50 |
        Format-Table -AutoSize
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to inventory applications on $ServerName : $($_.Exception.Message)"
}
```

### Search for Specific Application
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null
$SearchPattern = "*SQL*"  # e.g., "*Office*", "*Java*", "*Adobe*"

try {
    $result = Invoke-Command -ComputerName $ServerName -Credential $Credential -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        param($pattern)
        
        $registryPaths = @(
            'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
            'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
        )
        
        foreach ($path in $registryPaths) {
            Get-ItemProperty -Path $path -ErrorAction SilentlyContinue |
                Where-Object { $_.DisplayName -like $pattern } |
                Select-Object DisplayName, DisplayVersion, Publisher, InstallDate
        }
        
    } -ArgumentList $SearchPattern -ErrorAction Stop
    
    if ($result) {
        Write-Host "`nApplications matching '$SearchPattern':" -ForegroundColor Cyan
        $result | Sort-Object DisplayName -Unique | Format-Table -AutoSize
    } else {
        Write-Host "`nNo applications found matching '$SearchPattern'" -ForegroundColor Yellow
    }
    
    $result
    
} catch {
    Write-Error "Application search failed: $($_.Exception.Message)"
}
```

### Get Windows Updates / Hotfixes
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$DaysBack = 30

$scriptBlock = {
    param($days)
    
    try {
        # Get installed hotfixes
        $hotfixes = Get-HotFix -ErrorAction Stop
        
        # Filter recent updates
        $recentHotfixes = $hotfixes | Where-Object { 
            $_.InstalledOn -and $_.InstalledOn -gt (Get-Date).AddDays(-$days) 
        }
        
        [PSCustomObject]@{
            TotalHotfixes  = $hotfixes.Count
            AllHotfixes    = $hotfixes | Sort-Object InstalledOn -Descending
            RecentHotfixes = $recentHotfixes
        }
        
    } catch {
        throw "Hotfix enumeration failed: $($_.Exception.Message)"
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
    
    Write-Host "`nRetrieving Windows Updates/Hotfixes from $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Windows Update Summary ===" -ForegroundColor Yellow
    Write-Host "  Total Hotfixes Installed: $($result.TotalHotfixes)"
    Write-Host ""
    
    if ($result.RecentHotfixes) {
        Write-Host "=== Recent Updates (Last $DaysBack Days) ===" -ForegroundColor Yellow
        $result.RecentHotfixes | Format-Table HotFixID, Description, InstalledOn, InstalledBy -AutoSize
    } else {
        Write-Host "⚠ No updates installed in last $DaysBack days" -ForegroundColor Yellow
        Write-Host "  → Server may be missing critical security patches" -ForegroundColor Red
    }
    
    Write-Host "`n=== Last 20 Installed Updates ===" -ForegroundColor Yellow
    $result.AllHotfixes | Select-Object -First 20 | 
        Format-Table HotFixID, Description, InstalledOn, InstalledBy -AutoSize
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to retrieve hotfixes from $ServerName : $($_.Exception.Message)"
}
```

### Get Windows Features (Server Roles/Features)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Get installed Windows Features
        $features = Get-WindowsFeature -ErrorAction SilentlyContinue | Where-Object { $_.Installed }
        
        # Categorize features
        $roles = $features | Where-Object { $_.FeatureType -eq "Role" }
        $roleServices = $features | Where-Object { $_.FeatureType -eq "Role Service" }
        $featuresList = $features | Where-Object { $_.FeatureType -eq "Feature" }
        
        [PSCustomObject]@{
            TotalInstalled   = $features.Count
            Roles            = $roles
            RoleServices     = $roleServices
            Features         = $featuresList
            AllFeatures      = $features
        }
        
    } catch {
        # Get-WindowsFeature may not be available (non-Server SKU or missing module)
        [PSCustomObject]@{
            TotalInstalled = 0
            Error          = "Windows Features enumeration not available on this system"
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
    
    Write-Host "`nRetrieving Windows Features from $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    if ($result.Error) {
        Write-Host "`n$($result.Error)" -ForegroundColor Yellow
    } else {
        Write-Host "`n=== Windows Features Summary ===" -ForegroundColor Yellow
        Write-Host "  Total Installed: $($result.TotalInstalled)"
        Write-Host "  Roles:           $($result.Roles.Count)"
        Write-Host "  Role Services:   $($result.RoleServices.Count)"
        Write-Host "  Features:        $($result.Features.Count)"
        Write-Host ""
        
        if ($result.Roles) {
            Write-Host "=== Installed Roles ===" -ForegroundColor Yellow
            $result.Roles | Format-Table Name, DisplayName -AutoSize
        }
        
        if ($result.Features | Select-Object -First 1) {
            Write-Host "=== Installed Features (Top 20) ===" -ForegroundColor Yellow
            $result.Features | Select-Object -First 20 | Format-Table Name, DisplayName -AutoSize
        }
    }
    
    $result
    
} catch {
    Write-Error "Failed to retrieve Windows Features: $($_.Exception.Message)"
}
```

### Get Software from Win32_Product (Alternative Method)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

Write-Host "Querying Win32_Product (this may take several minutes)..." -ForegroundColor Yellow

$scriptBlock = {
    try {
        # Note: Win32_Product is slow and triggers consistency checks
        # Use registry method when possible
        $products = Get-CimInstance -ClassName Win32_Product -ErrorAction Stop
        
        $products | Select-Object Name, Version, Vendor, InstallDate |
            Sort-Object Name
        
    } catch {
        throw "Win32_Product query failed: $($_.Exception.Message)"
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
    
    $products = Invoke-Command @invokeParams
    
    Write-Host "`nInstalled Products (via Win32_Product):" -ForegroundColor Cyan
    Write-Host "Total: $($products.Count)" -ForegroundColor Yellow
    $products | Format-Table Name, Version, Vendor -AutoSize
    
    $products
    
} catch {
    Write-Error "Win32_Product query failed: $($_.Exception.Message)"
}
```

## Interpreting Results

### Install Date Analysis

| Finding | Meaning | Action |
|---------|---------|--------|
| Many recent installs | Software updates, new deployments | Normal if expected |
| No recent installs (>90 days) | No patching or updates | Check update process |
| Install after issue started | New software may be cause | Investigate |

### Common Application Categories

**Microsoft**
- SQL Server, Exchange, SharePoint, Office, .NET Framework, Visual C++ Redistributables

**Java**
- Multiple versions common; outdated versions are security risks

**Adobe**
- Reader, Flash (legacy - should be removed), AIR

**Antivirus/Security**
- Should be present and up-to-date

**Monitoring Agents**
- SCOM, monitoring tools, backup agents

### Problematic Software Indicators

| Issue | Risk | Action |
|-------|------|--------|
| Multiple Java versions | Security vulnerability | Remove old versions |
| Flash Player installed | End-of-life software | Uninstall immediately |
| Outdated software | Security/compatibility | Update or remove |
| Unknown publishers | Potential malware | Investigate |
| Duplicate software | Wasted resources | Clean up |

### Windows Updates

**Healthy**: Regular monthly updates (Patch Tuesday = 2nd Tuesday of month)
**Warning**: No updates >60 days
**Critical**: No updates >90 days (missing security patches)

### Version Tracking
Track critical software versions:
- **SQL Server**: Check against support lifecycle
- **.NET Framework**: Multiple versions expected
- **PowerShell**: Version 5.1+ recommended
- **Server role software**: Keep current

## Common Issues

### Missing Critical Software
- No antivirus/security software
- No monitoring agents (if managed)
- Missing runtime dependencies (.NET, Visual C++ Redistributables)

### Outdated Software
- Old Java versions (security risk)
- Adobe Flash (end-of-life, remove immediately)
- Out-of-support products

### Installation Problems
- Failed installations (partial entries in registry)
- Duplicate/conflicting versions
- Corrupted installations

## Error Handling

```powershell
# Handle missing registry keys or permissions
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        $apps = @()
        
        $paths = @(
            'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
            'HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
        )
        
        foreach ($path in $paths) {
            try {
                $apps += Get-ItemProperty -Path $path -ErrorAction SilentlyContinue |
                    Where-Object { $_.DisplayName } |
                    Select-Object DisplayName, DisplayVersion
            } catch {
                # Path doesn't exist or access denied; continue
            }
        }
        
        $apps | Sort-Object DisplayName -Unique
        
    } -ErrorAction Stop
    
    $result | Format-Table -AutoSize
    
} catch {
    Write-Error "Application inventory failed: $($_.Exception.Message)"
}
```

## Performance Note

**Win32_Product Class**: Querying Win32_Product is SLOW (can take 5-10 minutes) and triggers Windows Installer consistency checks. Use the registry-based method instead unless you specifically need Win32_Product data.

## Next Steps

Based on installed applications:
- **Outdated software** → Plan updates or removal
- **Security software missing** → Install and configure
- **No recent updates** → Check **services** for Windows Update service
- **Unknown software** → Investigate with **processes** and event logs
- **Failed installations** → Check **event-logs** for installation errors
