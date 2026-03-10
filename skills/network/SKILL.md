# Network Configuration and Connectivity

## Purpose
Analyze network configuration including IP addresses, DNS settings, open ports, firewall rules, network adapter status, and connectivity issues.

## PowerShell Code

### Complete Network Configuration Analysis
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Network Adapters
        $adapters = Get-NetAdapter | Where-Object { $_.Status -ne "Disabled" }
        
        $adapterInfo = foreach ($adapter in $adapters) {
            # IP Configuration
            $ipConfig = Get-NetIPAddress -InterfaceIndex $adapter.InterfaceIndex -ErrorAction SilentlyContinue
            $ipv4 = $ipConfig | Where-Object { $_.AddressFamily -eq "IPv4" }
            $ipv6 = $ipConfig | Where-Object { $_.AddressFamily -eq "IPv6" -and $_.IPAddress -notlike "fe80:*" }
            
            # DNS Servers
            $dnsServers = Get-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ErrorAction SilentlyContinue
            $dnsIPv4 = ($dnsServers | Where-Object { $_.AddressFamily -eq 2 }).ServerAddresses -join ", "
            
            # Default Gateway
            $gateway = Get-NetRoute -InterfaceIndex $adapter.InterfaceIndex -DestinationPrefix "0.0.0.0/0" -ErrorAction SilentlyContinue
            
            [PSCustomObject]@{
                Name            = $adapter.Name
                Description     = $adapter.InterfaceDescription
                Status          = $adapter.Status
                LinkSpeed       = $adapter.LinkSpeed
                MacAddress      = $adapter.MacAddress
                IPv4Address     = $ipv4.IPAddress -join ", "
                IPv4PrefixLength = $ipv4.PrefixLength
                IPv6Address     = $ipv6.IPAddress -join ", "
                DNSServers      = $dnsIPv4
                DefaultGateway  = $gateway.NextHop -join ", "
                DHCPEnabled     = (Get-NetIPInterface -InterfaceIndex $adapter.InterfaceIndex -AddressFamily IPv4).Dhcp
            }
        }
        
        # DNS Configuration
        $dnsCache = Get-DnsClientCache -ErrorAction SilentlyContinue | Select-Object -First 20
        
        # Routing Table (key routes)
        $routes = Get-NetRoute | Where-Object { 
            $_.DestinationPrefix -eq "0.0.0.0/0" -or 
            $_.DestinationPrefix -like "10.*" -or 
            $_.DestinationPrefix -like "172.*" -or 
            $_.DestinationPrefix -like "192.168.*"
        }
        
        [PSCustomObject]@{
            Adapters        = $adapterInfo
            DNSCache        = $dnsCache
            Routes          = $routes
            ComputerName    = $env:COMPUTERNAME
            Timestamp       = Get-Date
        }
        
    } catch {
        throw "Network analysis failed: $($_.Exception.Message)"
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
    
    Write-Host "`nAnalyzing network configuration on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Network Adapters ===" -ForegroundColor Yellow
    $result.Adapters | Format-Table Name, Status, IPv4Address, DNSServers, DefaultGateway, LinkSpeed -AutoSize
    
    # Check for issues
    $downAdapters = $result.Adapters | Where-Object { $_.Status -ne "Up" }
    $noDNS = $result.Adapters | Where-Object { -not $_.DNSServers -and $_.Status -eq "Up" }
    $noGateway = $result.Adapters | Where-Object { -not $_.DefaultGateway -and $_.Status -eq "Up" }
    
    if ($downAdapters) {
        Write-Host "`n⚠ Adapters Not Up:" -ForegroundColor Yellow
        $downAdapters | Format-Table Name, Status -AutoSize
    }
    
    if ($noDNS) {
        Write-Host "`n⚠ Adapters Missing DNS Configuration:" -ForegroundColor Yellow
        $noDNS | Format-Table Name, IPv4Address -AutoSize
    }
    
    if ($noGateway) {
        Write-Host "`n⚠ Adapters Missing Default Gateway:" -ForegroundColor Yellow
        $noGateway | Format-Table Name, IPv4Address -AutoSize
    }
    
    Write-Host "`n=== Key Routes ===" -ForegroundColor Yellow
    $result.Routes | Format-Table DestinationPrefix, NextHop, InterfaceAlias, RouteMetric -AutoSize
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to analyze network on $ServerName : $($_.Exception.Message)"
}
```

### Test Network Connectivity (Ping, Port Tests)
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$TargetHost = "8.8.8.8"  # Test target (e.g., DNS server, gateway, internet)
$TargetPorts = @(80, 443, 3389, 445, 53)  # Ports to test

$scriptBlock = {
    param($target, $ports)
    
    # Ping test
    $pingResult = Test-Connection -ComputerName $target -Count 4 -ErrorAction SilentlyContinue
    $pingSuccess = $pingResult -ne $null
    $avgLatency = if ($pingSuccess) { 
        [math]::Round(($pingResult | Measure-Object ResponseTime -Average).Average, 2) 
    } else { 
        $null 
    }
    
    # Port tests
    $portResults = foreach ($port in $ports) {
        $tcpClient = New-Object System.Net.Sockets.TcpClient
        try {
            $connect = $tcpClient.BeginConnect($target, $port, $null, $null)
            $wait = $connect.AsyncWaitHandle.WaitOne(3000, $false)
            
            if ($wait) {
                $tcpClient.EndConnect($connect)
                $open = $true
            } else {
                $open = $false
            }
        } catch {
            $open = $false
        } finally {
            $tcpClient.Close()
        }
        
        [PSCustomObject]@{
            Port   = $port
            Status = if ($open) { "Open" } else { "Closed/Filtered" }
        }
    }
    
    [PSCustomObject]@{
        Target        = $target
        PingSuccess   = $pingSuccess
        AvgLatencyMs  = $avgLatency
        PacketLoss    = if ($pingSuccess) { 0 } else { 100 }
        PortTests     = $portResults
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($TargetHost, $TargetPorts)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nTesting connectivity from $ServerName to $TargetHost..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Connectivity Test Results ===" -ForegroundColor Yellow
    Write-Host "Target:          $($result.Target)"
    Write-Host "Ping Success:    $($result.PingSuccess)"
    if ($result.PingSuccess) {
        Write-Host "Avg Latency:     $($result.AvgLatencyMs) ms" -ForegroundColor Green
    } else {
        Write-Host "Ping Failed:     Host unreachable or ICMP blocked" -ForegroundColor Red
    }
    Write-Host ""
    
    Write-Host "=== Port Tests ===" -ForegroundColor Yellow
    $result.PortTests | Format-Table Port, Status -AutoSize
    
    $closedPorts = $result.PortTests | Where-Object { $_.Status -ne "Open" }
    if ($closedPorts) {
        Write-Host "`n⚠ Some ports are closed or filtered - may indicate firewall blocking" -ForegroundColor Yellow
    }
    
    $result
    
} catch {
    Write-Error "Connectivity test failed: $($_.Exception.Message)"
}
```

### Get Listening Ports and Connections
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)

$scriptBlock = {
    try {
        # Get listening TCP ports
        $listening = Get-NetTCPConnection -State Listen -ErrorAction Stop |
            Select-Object LocalAddress, LocalPort, 
                @{L='Process';E={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}},
                OwningProcess |
            Sort-Object LocalPort
        
        # Get established connections
        $established = Get-NetTCPConnection -State Established -ErrorAction Stop |
            Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State,
                @{L='Process';E={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}},
                OwningProcess
        
        [PSCustomObject]@{
            ListeningPorts        = $listening
            EstablishedConnections = $established
            ListeningCount        = $listening.Count
            EstablishedCount      = $established.Count
        }
        
    } catch {
        throw "Port enumeration failed: $($_.Exception.Message)"
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
    
    Write-Host "`nEnumerating network connections on $ServerName..." -ForegroundColor Cyan
    $result = Invoke-Command @invokeParams
    
    Write-Host "`n=== Listening Ports ===" -ForegroundColor Yellow
    Write-Host "Total: $($result.ListeningCount)" -ForegroundColor Gray
    $result.ListeningPorts | Format-Table LocalAddress, LocalPort, Process, OwningProcess -AutoSize
    
    Write-Host "`n=== Established Connections (Top 20) ===" -ForegroundColor Yellow
    Write-Host "Total: $($result.EstablishedCount)" -ForegroundColor Gray
    $result.EstablishedConnections | Select-Object -First 20 |
        Format-Table LocalAddress, LocalPort, RemoteAddress, RemotePort, Process -AutoSize
    
    # Return full result
    $result
    
} catch {
    Write-Error "Failed to enumerate connections: $($_.Exception.Message)"
}
```

### Get Firewall Rules
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$Direction = "Inbound"  # Inbound or Outbound

$scriptBlock = {
    param($direction)
    
    try {
        # Get enabled firewall rules for specified direction
        $rules = Get-NetFirewallRule -Direction $direction -Enabled True -ErrorAction Stop
        
        $ruleDetails = foreach ($rule in $rules) {
            $portFilter = Get-NetFirewallPortFilter -AssociatedNetFirewallRule $rule -ErrorAction SilentlyContinue
            $appFilter = Get-NetFirewallApplicationFilter -AssociatedNetFirewallRule $rule -ErrorAction SilentlyContinue
            
            [PSCustomObject]@{
                Name        = $rule.Name
                DisplayName = $rule.DisplayName
                Enabled     = $rule.Enabled
                Direction   = $rule.Direction
                Action      = $rule.Action
                Protocol    = $portFilter.Protocol
                LocalPort   = $portFilter.LocalPort
                RemotePort  = $portFilter.RemotePort
                Program     = $appFilter.Program
            }
        }
        
        $ruleDetails | Sort-Object DisplayName
        
    } catch {
        throw "Firewall rule enumeration failed: $($_.Exception.Message)"
    }
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = @($Direction)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nRetrieving $Direction firewall rules from $ServerName..." -ForegroundColor Cyan
    $rules = Invoke-Command @invokeParams
    
    Write-Host "`n=== Enabled $Direction Firewall Rules (Top 30) ===" -ForegroundColor Yellow
    Write-Host "Total: $($rules.Count)" -ForegroundColor Gray
    $rules | Select-Object -First 30 | 
        Format-Table DisplayName, Action, Protocol, LocalPort, Program -AutoSize -Wrap
    
    # Analyze
    $blockRules = $rules | Where-Object { $_.Action -eq "Block" }
    Write-Host "`nBlock Rules: $($blockRules.Count)" -ForegroundColor Yellow
    
    $rules
    
} catch {
    Write-Error "Failed to retrieve firewall rules: $($_.Exception.Message)"
}
```

### DNS Resolution Test
```powershell
$ServerName = "TARGET_SERVER"
# For current user (default): no credential needed
# For explicit credentials: check if $credential variable exists (user must create BEFORE running gh copilot)
$HostnamesToResolve = @("google.com", "dc01.contoso.com", "fileserver01")  # Test hostnames

$scriptBlock = {
    param($hostnames)
    
    $results = foreach ($hostname in $hostnames) {
        try {
            $resolved = Resolve-DnsName -Name $hostname -ErrorAction Stop
            $ipAddress = ($resolved | Where-Object { $_.Type -eq "A" }).IPAddress -join ", "
            
            [PSCustomObject]@{
                Hostname    = $hostname
                Resolved    = $true
                IPAddress   = $ipAddress
                Type        = ($resolved | Select-Object -First 1).Type
                Error       = $null
            }
        } catch {
            [PSCustomObject]@{
                Hostname    = $hostname
                Resolved    = $false
                IPAddress   = $null
                Type        = $null
                Error       = $_.Exception.Message
            }
        }
    }
    
    $results
}

try {
    $invokeParams = @{
        ComputerName = $ServerName
        ScriptBlock  = $scriptBlock
        ArgumentList = (,$HostnamesToResolve)
        ErrorAction  = 'Stop'
    UseSSL        = $true
    Port          = 5986
    SessionOption = (New-PSSessionOption -SkipCACheck -SkipCNCheck)
    }
    
    if ($credential) {
        $invokeParams['Credential'] = $credential
    }
    
    Write-Host "`nTesting DNS resolution from $ServerName..." -ForegroundColor Cyan
    $results = Invoke-Command @invokeParams
    
    Write-Host "`n=== DNS Resolution Results ===" -ForegroundColor Yellow
    $results | Format-Table Hostname, Resolved, IPAddress, Error -AutoSize
    
    $failed = $results | Where-Object { -not $_.Resolved }
    if ($failed) {
        Write-Host "`n⚠ DNS resolution failed for:" -ForegroundColor Red
        $failed | Format-Table Hostname, Error -AutoSize
        Write-Host "  → Check DNS server configuration and network connectivity" -ForegroundColor Yellow
    } else {
        Write-Host "`n✓ All DNS resolutions successful" -ForegroundColor Green
    }
    
    $results
    
} catch {
    Write-Error "DNS resolution test failed: $($_.Exception.Message)"
}
```

## Interpreting Results

### Network Adapter Status

| Status | Meaning | Action |
|--------|---------|--------|
| Up | Adapter is connected and active | Normal |
| Disconnected | Cable unplugged or wireless disconnected | Check physical connection |
| Disabled | Adapter administratively disabled | Enable if needed |

### Common Port Numbers

| Port | Service | Purpose |
|------|---------|---------|
| 22 | SSH | Secure shell |
| 53 | DNS | Domain name resolution |
| 80 | HTTP | Web traffic |
| 443 | HTTPS | Secure web traffic |
| 445 | SMB | File sharing |
| 3389 | RDP | Remote Desktop |
| 1433 | SQL | SQL Server |
| 5986 | WinRM | PowerShell remoting |

### Network Issues

| Finding | Likely Problem | Action |
|---------|---------------|--------|
| No DNS servers | DNS misconfiguration | Configure DNS |
| No default gateway | Cannot route off-subnet | Configure gateway |
| Adapter down | Physical/driver issue | Check cable, driver |
| High latency (>100ms) | Network congestion, routing | Investigate path |
| Ports closed | Firewall blocking | Check firewall rules |
| DNS resolution fails | DNS server issue, firewall | Check DNS settings |
| No established connections | Network isolation | Check firewall, routing |

### Connectivity Patterns

**Healthy Server**:
- All adapters "Up"
- DNS configured
- Default gateway present
- Can resolve internal/external DNS
- Expected ports listening

**Problem Indicators**:
- Adapter disconnected
- Missing DNS/gateway
- Cannot resolve hostnames
- Unexpected ports open (security)
- No connections (isolated)

## Error Handling

```powershell
# Handle missing network cmdlets (older PowerShell)
$ServerName = "TARGET_SERVER"

try {
    $result = Invoke-Command -ComputerName $ServerName -UseSSL -Port 5986 -SessionOption (New-PSSessionOption -SkipCACheck -SkipCNCheck) -ScriptBlock {
        try {
            # Try modern cmdlets
            Get-NetAdapter | Select-Object Name, Status, LinkSpeed
        } catch {
            # Fallback to WMI
            Get-CimInstance Win32_NetworkAdapter | Where-Object { $_.NetEnabled } |
                Select-Object Name, NetConnectionStatus, Speed
        }
    } -ErrorAction Stop
    
    $result | Format-Table -AutoSize
    
} catch {
    Write-Error "Network enumeration failed: $($_.Exception.Message)"
}
```

## Next Steps

Based on network analysis:
- **DNS issues** → Check **services** for DNS Client service
- **Adapter down** → Check physical connection, driver, **event-logs**
- **Port blocked** → Check firewall rules, **services** for application
- **High latency** → Check **performance** for network bottleneck
- **Connection issues** → Test with **connectivity** checks, check routing
