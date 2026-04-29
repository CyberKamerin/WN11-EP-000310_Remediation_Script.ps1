# 🛡️ WN11-EP-000310 Remediation Script

## 👤 Author
**Kamerin Crawford**  
GitHub: https://github.com/CyberKamerin  

---

## 📌 Overview
This PowerShell script remediates **WN11-EP-000310** by enforcing:

- Enumeration policy for external devices incompatible with Kernel DMA Protection  
- **Policy সেট to: Block All**

The script is idempotent and logs all actions.

---

## ⚙️ PowerShell Script

```powershell
$LogDirectory = "C:\Logs"
$LogFile = Join-Path $LogDirectory "PolicyHardening.log"

$RegPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Kernel DMA Protection"
$ValueName = "DeviceEnumerationPolicy"
$DesiredValue = 0  # 0 = Block All

# Create log directory if missing
if (!(Test-Path $LogDirectory)) {
    New-Item -ItemType Directory -Path $LogDirectory -Force | Out-Null
}

function Write-Log {
    param(
        [string]$Message,
        [ConsoleColor]$Color = "White"
    )

    $Timestamp = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    $Entry = "$Timestamp - $Message"

    Write-Host $Entry -ForegroundColor $Color
    Add-Content -Path $LogFile -Value $Entry
}

# Admin check
if (-not ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
    [Security.Principal.WindowsBuiltInRole]::Administrator)) {

    Write-Log "ERROR: Script must be run as Administrator." Red
    exit 1
}

Write-Log "Starting STIG enforcement: WN11-EP-000310"

# Ensure registry path exists
if (-not (Test-Path $RegPath)) {
    Write-Log "Creating registry path: $RegPath"

    try {
        New-Item -Path $RegPath -Force | Out-Null
        Write-Log "Registry path created successfully." Green
    }
    catch {
        Write-Log "ERROR: Failed to create registry path. $_" Red
        exit 1
    }
}
else {
    Write-Log "Registry path already exists" Cyan
}

# Read current value
try {
    $CurrentValue = (Get-ItemProperty -Path $RegPath -Name $ValueName -ErrorAction Stop).$ValueName
    Write-Log "Current value: $CurrentValue" Cyan
}
catch {
    $CurrentValue = $null
    Write-Log "Value not set" Yellow
}

# Apply remediation if needed
if ($CurrentValue -ne $DesiredValue) {

    Write-Log "Applying policy (Block All)..."

    try {
        New-ItemProperty -Path $RegPath -Name $ValueName -Value $DesiredValue -PropertyType DWord -Force | Out-Null
        Write-Log "Policy applied" Green
    }
    catch {
        Write-Log "ERROR: Failed to apply policy. $_" Red
        exit 1
    }

}
else {
    Write-Log "Already compliant" Cyan
}

# Verify
try {
    $Verify = (Get-ItemProperty -Path $RegPath -Name $ValueName -ErrorAction Stop).$ValueName

    if ($Verify -eq $DesiredValue) {
        Write-Log "Verification PASSED" Green
    }
    else {
        Write-Log "Verification FAILED" Red
        exit 1
    }
}
catch {
    Write-Log "Verification FAILED: Unable to read value" Red
    exit 1
}

# Refresh Group Policy
Write-Log "Running gpupdate..."

try {
    gpupdate /force | Out-Null
    Write-Log "Group Policy refreshed" Green
}
catch {
    Write-Log "WARNING: gpupdate issue" Yellow
}

Write-Log "Completed remediation" Green
