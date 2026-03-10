---
name: roles-features
description: "Check installed Windows Server roles and features"
---

# Windows Server Roles and Features

## Purpose
Inventory installed Windows Server roles, role services, and features to understand server configuration and identify potential issues.

## PowerShell Code

### Complete Roles and Features Inventory
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Check if Server Manager module is available
        $serverManagerAvailable = Get-Module -ListAvailable -Name ServerManager -ErrorAction SilentlyContinue
        
        if (-not $serverManagerAvailable) {
            throw "ServerManager module not available. This may not be a Windows Server system or the module is not installed."
        }
        
        Import-Module ServerManager -ErrorAction Stop
        
        # Get all Windows Features
        $allFeatures = Get-WindowsFeature -ErrorAction Stop
        
        # Installed features
        $installed = $allFeatures | Where-Object { $_.Installed }
        
        # Categorize installed features
        $roles = $installed | Where-Object { $_.FeatureType -eq "Role" }
        $roleServices = $installed | Where-Object { $_.FeatureType -eq "Role Service" }
        $features = $installed | Where-Object { $_.FeatureType -eq "Feature" }
        
        # Available but not installed (for reference)
        $available = $allFeatures | Where-Object { -not $_.Installed -and $_.InstallState -ne "Removed" }
        
        [PSCustomObject]@{
            ServerName           = $env:COMPUTERNAME
            TotalInstalled       = $installed.Count
            RolesInstalled       = $roles.Count
            RoleServicesInstalled = $roleServices.Count
            FeaturesInstalled    = $features.Count
            AvailableCount       = $available.Count
            
            Roles                = $roles
            RoleServices         = $roleServices
            Features             = $features
            AllInstalled         = $installed
            AllAvailable         = $available
            
            Timestamp            = Get-Date
        }
        
    } catch {
        throw "Roles and features enumeration failed: $($_.Exception.Message)"
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
    
    Write-Host "`nInventorying Windows Server Roles and Features on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Roles and Features Summary ===" -ForegroundColor Yellow
    Write-Host "  Total Installed:     $($result.TotalInstalled)"
    Write-Host "  Roles:               $($result.RolesInstalled)"
    Write-Host "  Role Services:       $($result.RoleServicesInstalled)"
    Write-Host "  Features:            $($result.FeaturesInstalled)"
    Write-Host ""
    
    if ($result.Roles) {
        Write-Host "=== Installed Roles ===" -ForegroundColor Yellow
        $result.Roles | Format-Table Name, DisplayName, InstallState -AutoSize
    } else {
        Write-Host "No server roles installed (workgroup or minimal install)" -ForegroundColor Gray
    }
    
    if ($result.RoleServices) {
        Write-Host "`n=== Installed Role Services (Top 20) ===" -ForegroundColor Yellow
        $result.RoleServices | Select-Object -First 20 |
            Format-Table Name, DisplayName, InstallState -AutoSize
    }
    
    if ($result.Features) {
        Write-Host "`n=== Installed Features (Top 20) ===" -ForegroundColor Yellow
        $result.Features | Select-Object -First 20 |
            Format-Table Name, DisplayName, InstallState -AutoSize
    }
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to inventory roles and features on $ServerName : $($_.Exception.Message)"
    
    if ($_.Exception.Message -match "ServerManager module not available") {
        Write-Host "`nNote: This system may not be Windows Server, or ServerManager module is missing" -ForegroundColor Yellow
    }
}
```

### Check for Specific Role or Feature
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$FeatureName = "Web-Server"  # e.g., "Web-Server", "AD-Domain-Services", "RSAT-AD-Tools"

$scriptBlock = {
    param($name)
    
    try {
        Import-Module ServerManager -ErrorAction Stop
        
        $feature = Get-WindowsFeature -Name $name -ErrorAction Stop
        
        [PSCustomObject]@{
            Name        = $feature.Name
            DisplayName = $feature.DisplayName
            Installed   = $feature.Installed
            InstallState = $feature.InstallState
            FeatureType = $feature.FeatureType
            Path        = $feature.Path
        }
        
    } catch {
        throw "Feature check failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($FeatureName)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nChecking for feature '$FeatureName' on $ServerName..." -ForegroundColor Cyan
    $feature = Invoke-Command @invokeParams
    
    Write-Host "`n=== Feature Status ===" -ForegroundColor Yellow
    Write-Host "Name:         $($feature.Name)"
    Write-Host "Display Name: $($feature.DisplayName)"
    Write-Host "Type:         $($feature.FeatureType)"
    Write-Host "Installed:    $($feature.Installed)" -ForegroundColor $(if ($feature.Installed) { "Green" } else { "Yellow" })
    Write-Host "State:        $($feature.InstallState)"
    
    $feature
    
} catch {
    Write-Error "Failed to check feature: $($_.Exception.Message)"
}
```

### Get Role-Specific Configuration (IIS Example)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Check if IIS (Web-Server) role is installed
        Import-Module ServerManager -ErrorAction Stop
        $iisInstalled = (Get-WindowsFeature -Name Web-Server).Installed
        
        if (-not $iisInstalled) {
            throw "IIS (Web-Server) role is not installed"
        }
        
        # Get IIS-related features
        $iisFeatures = Get-WindowsFeature -Name "Web-*" | Where-Object { $_.Installed }
        
        # Get IIS sites (requires WebAdministration module)
        Import-Module WebAdministration -ErrorAction Stop
        $sites = Get-Website -ErrorAction Stop
        
        $siteInfo = foreach ($site in $sites) {
            [PSCustomObject]@{
                Name            = $site.Name
                ID              = $site.Id
                State           = $site.State
                PhysicalPath    = $site.PhysicalPath
                ApplicationPool = $site.ApplicationPool
                Bindings        = ($site.Bindings.Collection | ForEach-Object { $_.BindingInformation }) -join ", "
            }
        }
        
        # Get Application Pools
        $appPools = Get-ChildItem IIS:\AppPools -ErrorAction Stop
        
        $appPoolInfo = foreach ($pool in $appPools) {
            [PSCustomObject]@{
                Name         = $pool.Name
                State        = $pool.State
                ManagedPipelineMode = $pool.ManagedPipelineMode
                RuntimeVersion = $pool.ManagedRuntimeVersion
                Identity     = $pool.ProcessModel.IdentityType
            }
        }
        
        [PSCustomObject]@{
            IISInstalled     = $iisInstalled
            IISFeatures      = $iisFeatures
            Sites            = $siteInfo
            ApplicationPools = $appPoolInfo
        }
        
    } catch {
        throw "IIS configuration check failed: $($_.Exception.Message)"
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
    
    Write-Host "`nAnalyzing IIS configuration on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== IIS Configuration ===" -ForegroundColor Yellow
    Write-Host "IIS Installed: $($result.IISInstalled)" -ForegroundColor Green
    Write-Host ""
    
    Write-Host "=== IIS Features Installed ===" -ForegroundColor Yellow
    $result.IISFeatures | Format-Table Name, DisplayName -AutoSize
    
    Write-Host "`n=== IIS Websites ===" -ForegroundColor Yellow
    $result.Sites | Format-Table Name, State, ApplicationPool, PhysicalPath -AutoSize
    
    Write-Host "`n=== Application Pools ===" -ForegroundColor Yellow
    $result.ApplicationPools | Format-Table Name, State, RuntimeVersion, Identity -AutoSize
    
    $result
    
} catch {
    Write-Error "Failed to analyze IIS configuration: $($_.Exception.Message)"
}
```

### Get Active Directory Domain Services Configuration
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Check if AD DS role is installed
        Import-Module ServerManager -ErrorAction Stop
        $addsInstalled = (Get-WindowsFeature -Name AD-Domain-Services).Installed
        
        if (-not $addsInstalled) {
            throw "Active Directory Domain Services role is not installed"
        }
        
        # Get AD DS features
        $adFeatures = Get-WindowsFeature -Name "AD-*" | Where-Object { $_.Installed }
        
        # Get domain controller info (requires Active Directory module)
        Import-Module ActiveDirectory -ErrorAction Stop
        
        $domain = Get-ADDomain -ErrorAction Stop
        $forest = Get-ADForest -ErrorAction Stop
        $dcInfo = Get-ADDomainController -Identity $env:COMPUTERNAME -ErrorAction Stop
        
        [PSCustomObject]@{
            ADDSInstalled       = $addsInstalled
            ADFeatures          = $adFeatures
            
            DomainName          = $domain.DNSRoot
            NetBIOSName         = $domain.NetBIOSName
            DomainMode          = $domain.DomainMode
            ForestMode          = $forest.ForestMode
            
            DCHostname          = $dcInfo.HostName
            DCIPv4Address       = $dcInfo.IPv4Address
            IsGlobalCatalog     = $dcInfo.IsGlobalCatalog
            IsReadOnly          = $dcInfo.IsReadOnly
            OperatingSystem     = $dcInfo.OperatingSystem
            Site                = $dcInfo.Site
            
            FSMORoles           = @{
                SchemaMaster       = $forest.SchemaMaster
                DomainNamingMaster = $forest.DomainNamingMaster
                PDCEmulator        = $domain.PDCEmulator
                RIDMaster          = $domain.RIDMaster
                InfrastructureMaster = $domain.InfrastructureMaster
            }
        }
        
    } catch {
        throw "AD DS configuration check failed: $($_.Exception.Message)"
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
    
    Write-Host "`nAnalyzing Active Directory configuration on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Active Directory Domain Services ===" -ForegroundColor Yellow
    Write-Host "Domain:           $($result.DomainName)"
    Write-Host "NetBIOS:          $($result.NetBIOSName)"
    Write-Host "Domain Mode:      $($result.DomainMode)"
    Write-Host "Forest Mode:      $($result.ForestMode)"
    Write-Host ""
    
    Write-Host "=== Domain Controller Info ===" -ForegroundColor Yellow
    Write-Host "Hostname:         $($result.DCHostname)"
    Write-Host "IP Address:       $($result.DCIPv4Address)"
    Write-Host "Site:             $($result.Site)"
    Write-Host "Global Catalog:   $($result.IsGlobalCatalog)"
    Write-Host "Read-Only:        $($result.IsReadOnly)"
    Write-Host ""
    
    Write-Host "=== FSMO Roles ===" -ForegroundColor Yellow
    Write-Host "Schema Master:           $($result.FSMORoles.SchemaMaster)"
    Write-Host "Domain Naming Master:    $($result.FSMORoles.DomainNamingMaster)"
    Write-Host "PDC Emulator:            $($result.FSMORoles.PDCEmulator)"
    Write-Host "RID Master:              $($result.FSMORoles.RIDMaster)"
    Write-Host "Infrastructure Master:   $($result.FSMORoles.InfrastructureMaster)"
    
    $result
    
} catch {
    Write-Error "Failed to analyze AD DS: $($_.Exception.Message)"
}
```

## Interpreting Results

### Common Server Roles

| Role Name | Display Name | Purpose |
|-----------|--------------|---------|
| AD-Domain-Services | Active Directory Domain Services | Domain controller |
| DNS | DNS Server | Name resolution |
| DHCP | DHCP Server | IP address assignment |
| Web-Server | Web Server (IIS) | Web hosting |
| File-Services | File and Storage Services | File sharing |
| Print-Services | Print and Document Services | Print server |
| Hyper-V | Hyper-V | Virtualization |
| WDS | Windows Deployment Services | OS deployment |
| WSUS | Windows Server Update Services | Update management |

### Role Dependencies

Some roles require other roles/features:
- **AD DS** → DNS (typically)
- **IIS** → .NET Framework features
- **Remote Desktop Services** → Various RDS components
- **Failover Clustering** → Multiple nodes, shared storage

### Feature Categories

**Management Tools**
- RSAT-* (Remote Server Administration Tools)
- PowerShell modules
- GUI management consoles

**Network Features**
- Network Load Balancing
- SNMP Service
- Telnet Client/Server

**Core Features**
- .NET Framework (multiple versions)
- Windows PowerShell
- Background Intelligent Transfer Service (BITS)
- Windows Defender Features

### Common Issues by Role

**IIS (Web-Server)**
- Application pools stopped
- Permissions on physical paths
- Missing features (ASP.NET, URL Rewrite, etc.)

**Active Directory**
- Replication issues
- FSMO role holder unavailable
- DNS misconfiguration
- Time synchronization problems

**DNS Server**
- Zone transfer failures
- Forwarder issues
- Stale records

**File Services**
- Share permissions
- NTFS permissions
- Quota exceeded
- DFS replication lag

## Error Handling

```powershell
# Handle systems without ServerManager module
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        try {
            Import-Module ServerManager -ErrorAction Stop
            Get-WindowsFeature | Where-Object { $_.Installed } | 
                Select-Object Name, DisplayName, FeatureType
        } catch {
            # ServerManager not available
            [PSCustomObject]@{
                Error = "ServerManager module not available. This system may not be Windows Server."
            }
        }
    } -ErrorAction Stop
    
    if ($result.Error) {
        Write-Warning $result.Error
    } else {
        $result | Format-Table -AutoSize
    }
    
} catch {
    Write-Error "Roles enumeration failed: $($_.Exception.Message)"
}
```

## Role-Specific Diagnostics

### After identifying installed roles, run role-specific checks:

**IIS Web Server** → Check:
- Application pool status
- Website bindings
- SSL certificate validity
- URL Rewrite rules
- Logging configuration

**Active Directory** → Check:
- Replication status (`repadmin /replsummary`)
- FSMO role availability
- DNS health
- SYSVOL/NETLOGON shares

**DNS Server** → Check:
- Zone health
- Forwarders configured
- Event logs for DNS errors
- Query resolution

**File Server** → Check:
- Share permissions
- Disk space on volumes
- SMB version
- DFS namespace/replication status

## Next Steps

Based on roles and features:
- **Role installed** → Run role-specific diagnostics
- **IIS present** → Check **services** for W3SVC, **event-logs** for IIS events
- **AD DS present** → Verify replication, DNS, **event-logs** for AD events
- **No roles** → Verify server purpose, may be application server
- **Many features** → Ensure all are necessary (security best practice)
