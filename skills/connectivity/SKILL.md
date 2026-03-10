# Connectivity - Establish PowerShell Remoting Connection

## Purpose
Test connectivity to a Windows Server and establish a PowerShell remoting session. This is typically the first skill to run before any diagnostics.

## Prerequisites
- PowerShell remoting enabled on target server (WinRM HTTPS listener on port 5986)
- Appropriate credentials (current user by default, or explicit via Get-Credential for secure prompting)
- Network connectivity to target server on port 5986 (HTTPS)
- IP addresses are supported directly — no TrustedHosts modification required

## Credential Handling

⚠️ **SECURITY: NEVER ask users to type passwords in the chat.** Use `Get-Credential` which opens 
a secure Windows login dialog where passwords are never visible in the conversation.

**Default (current user):**
```powershell
$Credential = $null  # Uses current user's identity automatically
```

**Explicit credentials (secure dialog):**
```powershell
# Opens a Windows GUI dialog for secure password entry
$Credential = Get-Credential -UserName "domain\admin" -Message "Enter credentials for ServerName"
# User enters password in the dialog, NOT in the chat
```

## Connection Pattern

All connections use **HTTPS on port 5986** with `-SkipCACheck` and `-SkipCNCheck` session options. This works universally for domain-joined servers, workgroup servers, Azure VMs, and direct IP addresses — no TrustedHosts configuration needed.

## PowerShell Code

### Test Basic Connectivity
```powershell
# Test if server is reachable
$ServerName = "TARGET_SERVER"  # Hostname or IP address

Write-Host "Testing connectivity to $ServerName..." -ForegroundColor Cyan

# Test network connectivity
$pingResult = Test-Connection -ComputerName $ServerName -Count 2 -Quiet -ErrorAction SilentlyContinue

if ($pingResult) {
    Write-Host "✓ Server is responding to ping" -ForegroundColor Green
} else {
    Write-Warning "✗ Server is not responding to ping (ICMP may be blocked)"
}

# Test TCP connectivity to HTTPS WinRM port
$tcpTest = Test-NetConnection -ComputerName $ServerName -Port 5986 -WarningAction SilentlyContinue
if ($tcpTest.TcpTestSucceeded) {
    Write-Host "✓ Port 5986 (WinRM HTTPS) is reachable" -ForegroundColor Green
} else {
    Write-Warning "✗ Port 5986 is NOT reachable — check firewall rules"
    Write-Host "  → Ensure WinRM HTTPS listener is configured on the target" -ForegroundColor Yellow
    Write-Host "  → Check Windows Firewall allows inbound TCP 5986" -ForegroundColor Yellow
    Write-Host "  → For Azure VMs, check NSG inbound rules" -ForegroundColor Yellow
}

# Test WinRM over HTTPS
try {
    Test-WSMan -ComputerName $ServerName -UseSSL -ErrorAction Stop
    Write-Host "✓ WinRM HTTPS is responding" -ForegroundColor Green
} catch {
    Write-Warning "✗ WinRM HTTPS is not available: $($_.Exception.Message)"
    Write-Host "  Ensure WinRM HTTPS listener is configured on the target" -ForegroundColor Yellow
}
```

### Establish PSSession (Reusable Session)
```powershell
$ServerName = "TARGET_SERVER"  # Hostname or IP address

# For explicit credentials, open secure dialog (NEVER type passwords in chat):
# $Credential = Get-Credential -UserName "domain\admin" -Message "Enter credentials for $ServerName"
# For current user (default):
$Credential = $null

$SessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
$splat = @{
    ComputerName  = $ServerName
    UseSSL        = $true
    Port          = 5986
    SessionOption = $SessionOption
    ErrorAction   = 'Stop'
}
if ($Credential) { $splat['Credential'] = $Credential }

try {
    $session = New-PSSession @splat
    Write-Host "✓ PSSession established to $ServerName over HTTPS" -ForegroundColor Green

    # Verify the session
    $result = Invoke-Command -Session $session -ScriptBlock {
        [PSCustomObject]@{
            ComputerName = $env:COMPUTERNAME
            UserContext  = "$env:USERDOMAIN\$env:USERNAME"
            PSVersion    = $PSVersionTable.PSVersion.ToString()
            Timestamp    = Get-Date
        }
    }

    Write-Host "  Computer: $($result.ComputerName)" -ForegroundColor Gray
    Write-Host "  User Context: $($result.UserContext)" -ForegroundColor Gray
    Write-Host "  PowerShell: $($result.PSVersion)" -ForegroundColor Gray

    # Session is ready — pass $session to subsequent Invoke-Command calls

} catch {
    Write-Warning "✗ Connection failed: $($_.Exception.Message)"

    if ($_.Exception.Message -match "Access is denied") {
        Write-Host "  → Check credentials and permissions (use Get-Credential for explicit creds)" -ForegroundColor Yellow
    } elseif ($_.Exception.Message -match "cannot be resolved") {
        Write-Host "  → Check DNS resolution and network connectivity" -ForegroundColor Yellow
    } elseif ($_.Exception.Message -match "certificate") {
        Write-Host "  → Certificate issue — SkipCACheck and SkipCNCheck should handle this" -ForegroundColor Yellow
    } elseif ($_.Exception.Message -match "cannot connect") {
        Write-Host "  → Check WinRM HTTPS listener, firewall, and NSG rules" -ForegroundColor Yellow
    }
}
```

### One-Shot Invoke-Command (No Persistent Session)
```powershell
$ServerName = "TARGET_SERVER"

# For explicit credentials, open secure dialog:
# $Credential = Get-Credential -Message "Enter credentials for $ServerName"
# For current user:
$Credential = $null

$SessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
$invokeParams = @{
    ComputerName  = $ServerName
    UseSSL        = $true
    Port          = 5986
    SessionOption = $SessionOption
    ErrorAction   = 'Stop'
    ScriptBlock   = {
        [PSCustomObject]@{
            ComputerName = $env:COMPUTERNAME
            UserContext  = "$env:USERDOMAIN\$env:USERNAME"
            PSVersion    = $PSVersionTable.PSVersion.ToString()
            Timestamp    = Get-Date
        }
    }
}
if ($Credential) { $invokeParams['Credential'] = $Credential }

try {
    $result = Invoke-Command @invokeParams
    Write-Host "✓ Successfully connected to $ServerName" -ForegroundColor Green
    $result
} catch {
    Write-Warning "✗ Connection failed: $($_.Exception.Message)"
}
```

### Create Reusable CIM Session
```powershell
$ServerName = "TARGET_SERVER"
$Credential = $null  # Set to Get-Credential result if needed

try {
    $CimOption = New-CimSessionOption -UseSsl -SkipCACheck -SkipCNCheck
    $cimSplat = @{
        ComputerName  = $ServerName
        SessionOption = $CimOption
        Port          = 5986
        ErrorAction   = 'Stop'
    }
    if ($Credential) { $cimSplat['Credential'] = $Credential }

    $cimSession = New-CimSession @cimSplat
    Write-Host "✓ CIM session established to $ServerName over HTTPS" -ForegroundColor Green

    # Test the session
    $os = Get-CimInstance -CimSession $cimSession -ClassName Win32_OperatingSystem
    Write-Host "  OS: $($os.Caption)" -ForegroundColor Gray

    # Keep session for reuse in diagnostic skills
    # Clean up when done: Remove-CimSession -CimSession $cimSession

} catch {
    Write-Warning "✗ Failed to create CIM session: $($_.Exception.Message)"
    Write-Host "  → Ensure WinRM HTTPS listener is configured on port 5986" -ForegroundColor Yellow
}
```

## Interpreting Results

### Success Indicators
- ✓ Port 5986 reachable → Network layer and firewall are open
- ✓ WinRM HTTPS responding → PowerShell remoting over HTTPS is enabled
- ✓ Command execution → Authentication and authorization succeeded

### Common Failure Scenarios

| Error Message | Likely Cause | Resolution |
|--------------|--------------|------------|
| "Access is denied" | Insufficient permissions or wrong credentials | Use account with local admin rights; verify credentials in Get-Credential dialog |
| "Cannot be resolved" | DNS/Name resolution | Check hostname, try IP address directly |
| "WinRM cannot process the request" | WinRM HTTPS not configured | Configure WinRM HTTPS listener on target |
| "Connection timed out" | Firewall/NSG blocking port 5986 | Check firewall and NSG inbound rules for TCP 5986 |
| "Logon failure" | Bad credentials | Verify username/password entered in Get-Credential dialog |
| "Server certificate invalid" | Certificate issue | Handled by -SkipCACheck and -SkipCNCheck |
| "Negotiate authentication error" | Kerberos over internet | Use explicit -Credential parameter via Get-Credential |

## Security Considerations

1. **Always HTTPS** — All connections use port 5986 with SSL/TLS encryption
2. **Never type passwords in chat** — Use `Get-Credential` which opens a secure Windows dialog for password entry
3. **SkipCACheck / SkipCNCheck** — Bypasses certificate validation for self-signed certs; acceptable for known servers you control
4. **IP addresses supported** — Connect directly to IPs without TrustedHosts modification
5. **Credentials never stored** — PSCredential objects are used once and discarded
6. **For production** — Consider CA-issued certificates and removing Skip flags

## WinRM HTTPS Setup on Target

If the target server does not have a WinRM HTTPS listener, configure it:

```powershell
# Run ON the target server (via RDP, console, or Azure Run Command)

# Create a self-signed certificate
$cert = New-SelfSignedCertificate `
    -DnsName $env:COMPUTERNAME `
    -CertStoreLocation Cert:\LocalMachine\My `
    -NotAfter (Get-Date).AddYears(3)

# Create WinRM HTTPS listener
winrm create winrm/config/Listener?Address=*+Transport=HTTPS `
    "@{Hostname=`"$env:COMPUTERNAME`"; CertificateThumbprint=`"$($cert.Thumbprint)`"}"

# Open firewall for HTTPS WinRM
New-NetFirewallRule -DisplayName "WinRM HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 5986 -Action Allow

# Verify listener is active
winrm enumerate winrm/config/Listener
```

## Next Steps

Once connectivity is established:
1. Run **server-overview** skill for basic system information
2. Run specific diagnostic skills based on the issue being investigated
3. Reuse the connection pattern established here for all subsequent commands
