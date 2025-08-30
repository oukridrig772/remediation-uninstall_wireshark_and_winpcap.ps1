# remediation-uninstall_wireshark_and_winpcap.ps1

# Set the applications to uninstall.
# Use the exact DisplayName as found in "Add or remove programs" or the registry.
$AppsToUninstall = @(
    "Wireshark 2.2.1 (64-bit)",
    "WinPcap 4.1.3"
)

# Loop through each application and find its uninstall string
foreach ($AppName in $AppsToUninstall) {
    Write-Host "Searching for uninstall information for: $AppName"

    # Search for the uninstall key in both 64-bit and 32-bit registry paths
    $uninstallKey = Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" -ErrorAction SilentlyContinue | Where-Object {$_.DisplayName -eq $AppName}
    if (!$uninstallKey) {
        $uninstallKey = Get-ItemProperty -Path "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" -ErrorAction SilentlyContinue | Where-Object {$_.DisplayName -eq $AppName}
    }

    if ($uninstallKey) {
        $uninstallString = $uninstallKey.UninstallString
        
        if ($uninstallString -match "msiexec.exe") {
            # Handle MSI-based uninstalls
            $guid = [regex]::Match($uninstallString, '\{[0-9A-Fa-f]{8}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{12}\}').Value
            if ($guid) {
                Write-Host "  Found MSI GUID: $guid"
                Write-Host "  Starting silent uninstall for $AppName..."
                Start-Process msiexec.exe -ArgumentList "/X $guid /qn /norestart" -Wait -PassThru
                Write-Host "  Uninstall of $AppName completed."
            } else {
                Write-Warning "  Could not find GUID in uninstall string for $AppName. Skipping."
            }
        } else {
            # Handle EXE-based uninstalls (Wireshark often uses this)
            Write-Host "  Found uninstaller: $uninstallString"
            Write-Host "  Starting silent uninstall for $AppName..."
            # For Wireshark's uninstaller, the /S switch is typically used for a silent uninstall.
            # You might need to adjust the argument based on the specific application.
            Start-Process -FilePath $uninstallString -ArgumentList "/S" -Wait -PassThru
            Write-Host "  Uninstall of $AppName completed."
        }
    } else {
        Write-Warning "  $AppName not found in the list of installed programs. Skipping."
    }
}

Write-Host "All specified applications have been uninstalled."
Write-Host "Restarting the machine in 5 seconds..."

# Delay to give the user a chance to see the message.
Start-Sleep -Seconds 5

# Restart the computer automatically. The -Force parameter bypasses confirmation.
Restart-Computer -Force
