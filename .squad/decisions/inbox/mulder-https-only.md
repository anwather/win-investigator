# Decision: HTTPS-Only Connection Pattern (Universal)

**Author:** Mulder (Backend/PowerShell)  
**Date:** Current session  
**Directive from:** Anthony Watherston (owner)  
**Status:** Implemented

## Summary

All PowerShell remoting connections in win-investigator now use a single universal pattern:

- **HTTPS on port 5986** — the ONLY transport. HTTP/5985 removed entirely.
- **`-SkipCACheck -SkipCNCheck`** on all `PSSessionOption` and `CimSessionOption` — handles self-signed certs and IP address connections.
- **No TrustedHosts modification** — the Skip flags eliminate this requirement.
- **IP addresses supported directly** — connect to `10.0.0.5` or `20.100.50.25` with no extra setup.

## Rationale

1. **Simplicity** — One connection pattern for all scenarios (domain, workgroup, Azure, IP, hostname).
2. **Security** — HTTPS encrypts all traffic. No accidental HTTP exposure.
3. **No client modification** — TrustedHosts changes require admin on the client and persist after the session. SkipCA/SkipCN are session-scoped.
4. **IP address support** — TrustedHosts was the only reason IP connections failed. SkipCNCheck removes this barrier.

## Standard Patterns

### PSSession / Invoke-Command
```powershell
$SessionOption = New-PSSessionOption -SkipCACheck -SkipCNCheck
$splat = @{
    ComputerName  = $ServerName
    UseSSL        = $true
    Port          = 5986
    SessionOption = $SessionOption
}
if ($Credential) { $splat['Credential'] = $Credential }
$session = New-PSSession @splat
```

### CIM Session
```powershell
$CimOption = New-CimSessionOption -UseSsl -SkipCACheck -SkipCNCheck
$cimSplat = @{
    ComputerName  = $ServerName
    SessionOption = $CimOption
    Port          = 5986
}
if ($Credential) { $cimSplat['Credential'] = $Credential }
$cimSession = New-CimSession @cimSplat
```

### Test-WSMan
```powershell
Test-WSMan -ComputerName $ServerName -UseSSL
```

## Files Updated

All 10 diagnostic skills, both connectivity skills, copilot-instructions.md, agent definition, README, and docs files.

## Implications

- **All agents:** Follow this pattern in any new skill or code.
- **Skinner (testing):** Test scenarios should target port 5986 with HTTPS.
- **Target servers:** Must have WinRM HTTPS listener on port 5986 (see connectivity skill for setup instructions).
