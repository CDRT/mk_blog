---
date:
    created: 2026-04-16
    updated: 2026-04-20
authors:
    - Phil
categories:
    - "2026"
title: Changing the BIOS Supervisor Password with Intune and the Think BIOS Config Tool V2
---

Using the Think BIOS Config Tool V2 and the Lenovo.BIOS.Config PowerShell module to securely change the BIOS supervisor password via an Intune Win32 app.
<!-- more -->

## Overview

This guide walks through deploying a supervisor password change to Lenovo devices using the **Think BIOS Config Tool V2** and the **Lenovo.BIOS.Config** PowerShell module as an Intune Win32 app. The approach uses an encrypted password change file so the supervisor password is never exposed in plain text.

!!! note
    Previously created INI files from the original HTA-based Think BIOS Config Tool are **not compatible** with V2 due to changes in encryption methods. You must recreate the password change file using the new tool.

## Prerequisites

- Think BIOS Config Tool V2 (`ThinkBIOSConfigUI.ps1`) and the `Lenovo.BIOS.Config` module installed on an admin workstation to generate the password change file.
- A source directory for the Win32 app content (e.g., `C:\Source\IntuneWin32\TBCT`).
- The [Win32 Content Prep Tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool) (`IntuneWinAppUtil.exe`).
- An existing supervisor password set on target devices.

!!! warning
    The new Think BIOS Config Tool v2 does **_NOT_** set an initial supervisor password. The only way to programmatically set an initial password is boot to System Deployment Boot [Mode](https://docs.lenovocdrt.com/ref/bios/sdbm/) or leverage a product offering by Absolute called **Remote SVP**.

## Prepare the Lenovo.BIOS.Config Module and Script

Organize the module files and script into the source directory that will be packaged for deployment.

1. Create the source directory: `C:\Source\IntuneWin32\TBCT`
2. Use the **Save-Module** cmdlet to download the module into the source directory.

    ```powershell
    Save-Module -Name Lenovo.BIOS.Config -Path "C:\Source\IntuneWin32\TBCT"
    ```

3. Create a PowerShell script named `Update-SupervisorPassword.ps1` in the root of the source directory.

??? example "Update-SupervisorPassword.ps1"

    ```powershell
    $ErrorActionPreference = 'Stop'

    # === Log File ===
    $LogFile = "$env:ProgramData\Lenovo\ThinkBiosConfig\Logs\PasswordChange.log"
    $LogDir = Split-Path $LogFile -Parent
    if (-not (Test-Path $LogDir)) { New-Item -ItemType Directory -Path $LogDir -Force | Out-Null }

    # === Write-Log Function ===
    function Write-Log
    {
        param (
            [Parameter(Mandatory = $true)]
            [string]$Message,
            [ValidateSet('INFO', 'WARN', 'ERROR', 'SUCCESS')]
            [string]$Level = 'INFO'
        )
        $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
        $logEntry = "$timestamp [$Level]: $Message"
        Add-Content -Path $LogFile -Value $logEntry -Encoding UTF8
    }

    # === Initial log entry ===
    Write-Log "=== LENOVO PASSWORD CHANGE CONFIG INSTALL STARTED ===" 'INFO'

    # --- Paths ---
    $ModulePath = Join-Path $PSScriptRoot 'Lenovo.BIOS.Config\*\*.psd1'

    # --- Secret Key ---
    $SecretKey = "secretkey"  # Secret key for decryption - update as needed

    # -------------------------------------------------
    # 1. Load Lenovo BIOS Config Module
    # -------------------------------------------------
    try
    {
        $ModuleFile = Get-Item $ModulePath | Select-Object -First 1
        if (-not $ModuleFile) { throw "No .psd1 module found in $ModulePath" }

        Import-Module $ModuleFile.FullName -Force
        Write-Log "Loaded module: $($ModuleFile.Directory.Name)" 'SUCCESS'
    }
    catch
    {
        Write-Log "Failed to load Lenovo BIOS Config module: $($_.Exception.Message)" 'ERROR'
        exit 1
    }

    # -------------------------------------------------
    # 2. Verify Password Change File
    # -------------------------------------------------
    try
    {
        $ConfigFile = Get-ChildItem -Path $PSScriptRoot -Filter "*.ini" | Select-Object -First 1
        if (-not $ConfigFile) { throw "No .ini file found in $PSScriptRoot" }
        $ConfigFilePath = $ConfigFile.FullName
        Write-Log "Found password change file: $ConfigFilePath" 'INFO'
    }
    catch
    {
        Write-Log "Failed to find password change file: $($_.Exception.Message)" 'ERROR'
        exit 1
    }

    # -------------------------------------------------
    # 3. Import Password Change File
    # -------------------------------------------------
    try
    {
        Import-LnvPasswordChangeFile -ConfigFile $ConfigFilePath -Key $SecretKey | Out-Null
        Write-Log "Password change configuration imported successfully." 'SUCCESS'

        Write-Log "=== PASSWORD CHANGE CONFIG INSTALL COMPLETE (Exit 3010) ===" 'SUCCESS'
        exit 3010
    }
    catch
    {
        Write-Log "EXCEPTION during password change import: $($_.Exception.Message)" 'ERROR'
        exit 1
    }
    ```

!!! note
    Alternatively, you can add code to install the module from the PowerShell gallery but not everyone may want to do this. Hence, saving the module in the Win32 app for this example.

**Directory Structure**
```
C:\Source\IntuneWin32\TBCT\
    Lenovo.BIOS.Config\
        <ModuleVersion>\
            ├─ Lenovo.BIOS.Config.psd1
            ├─ Lenovo.BIOS.Config.psm1
            └─ ...
    Update-SupervisorPassword.ps1
    svp.ini
```

## Generate the Password Change File

The encrypted password change file is created using the Think BIOS Config Tool V2.

### Using the GUI

1. Launch `ThinkBIOSConfigUI.ps1` as Administrator.
2. In the **Actions** section, click **Change Password**.
3. Enter the current password, new password, and confirm new password.
4. Click **Create Password Change File**.
5. Enter a passphrase. This will serve as the encrypting key.
6. An `.ini` file will be generated at `%ProgramData%\Lenovo\ThinkBiosConfig\Output`
7. Copy the `.ini` file into `C:\Source\IntuneWin32\TBCT`.

!!! note
    If using the GUI, the generated INI file name defaults to **Password**. You can rename it, but the script will pick up whichever `.ini` file is present in the package directory.

### Using the CLI

The following commands can be used to generate the password change file.

```powershell
Import-Module Lenovo.BIOS.Config -Force

Export-LnvPasswordChangeFile -FileLocation C:\Sources\IntuneWin32\TBCT\svp.ini -Type svp -Verbose
```

You'll be prompted for an encrypting key (passphrase), the current supervisor password, and the new supervisor password you intend to set.

![](https://cdrt.github.io/mk_blog/img/2026/tbct_v2_intune_svp/image1.jpg)

## Package with the Win32 Content Prep Tool

With the source directory containing the module folder, the script, and the password change INI file, use the Win32 Content Prep [Tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool) to create the `.intunewin` package.

```dos
IntuneWinAppUtil.exe -c "C:\Source\IntuneWin32\TBCT" -s Update-SupervisorPassword.ps1 -o "C:\Source\IntuneWin32\" -q
```

| Flag | Value |
|------|-------|
| `-c` | Source folder: `C:\Source\IntuneWin32\TBCT` |
| `-s` | Setup file: `Update-SupervisorPassword.ps1` |
| `-o` | Output folder: `C:\Source\IntuneWin32\` |
| `-q` | Quiet mode |

This produces `Update-SupervisorPassword.intunewin` in `C:\Source\IntuneWin32\`.

## Create the Win32 App in Intune

### App Information

Add a new **Windows app (Win32)** in the [Intune admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps) and select the `.intunewin` package file to upload. Fill out the required fields and click **Next**.

### Program
| Setting | Value |
|---|---|
| Installer type | Command line |
| Install command | `%windir%\sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File ".\Update-SupervisorPassword.ps1"` |
| Uninstaller type | Command line |
| Uninstall command | `cmd.exe /c` |
| Install behavior | **System** |
| Device restart behavior | **Determine behavior based on return codes** |

### Requirements
| Setting | Value |
|---|---|
| Minimum operating system | Windows 11 21H2 |

#### Additional Requirement Rule # 1

This rule checks is if the device is in OOBE. Ideal for pre-provisioning.

- **Rule type:** Script
- **Script:** (save/add the script below. Slightly modified from [here](https://oofhours.com/2023/09/15/detecting-when-you-are-in-oobe/))
- **Run script as 32-bit process on 64-bit clients:** No
- **Return type:** Boolean
- **Operator:** Equals → **No**

??? example "Requirement Rule Script #1"

    ```powershell
    $Definition = @"

    using System;
    using System.Text;
    using System.Collections.Generic;
    using System.Runtime.InteropServices;

    namespace Api
    {
        public class Kernel32
        {
            [DllImport("kernel32.dll", CharSet = CharSet.Auto, SetLastError = true)]
            public static extern int OOBEComplete(ref int bIsOOBEComplete);
        }
    }
    "@

    Add-Type -TypeDefinition $Definition -Language CSharp

    $IsOOBEComplete = $false
    $appRequirement = [Api.Kernel32]::OOBEComplete([ref] $IsOOBEComplete)

    if ($IsOOBEComplete -eq '1') { return $true }
    else { return $false }
    ```

#### Additional Requirement Rule # 2 (Optional)

This rule checks the PasswordState value in the Lenovo_BiosPasswordSettings WMI class. If no password is set (value = 0), the requirement fails. The app will appear in Company Portal but will be greyed out to install if the assignment is set as available.

- **Rule type:** Script
- **Script:** (save/add the script below.)
- **Run script as 32-bit process on 64-bit clients:** No
- **Return type:** Boolean
- **Operator:** Equals → **Yes**

??? example "Requirement Rule Script #2"

    ```powershell
    $namespace = "root\wmi"
    $class = "Lenovo_BiosPasswordSettings"
    $value = (Get-CimInstance -Namespace $namespace -ClassName $class -ErrorAction SilentlyContinue).PasswordState

    if ($value -ne 0) { return $true } # Password is set
    else { return $false } # No password set
    ```

### Detection Rules

Add a **custom detection script** for the rule

??? example "Detection Script"

    ```powershell

    # Define the path where the log file is located
    $tbctPath = Join-Path -Path $env:ProgramData -ChildPath "Lenovo\ThinkBiosConfig"

    # Find the latest log file (*.log or *.txt) in the Lenovo ThinkBiosConfig directory
    $logFile = Get-ChildItem -Path $tbctPath -Filter "*.log" -Recurse -ErrorAction SilentlyContinue |
        Sort-Object -Property LastWriteTime -Descending |
        Select-Object -Last 1

    if ($logFile)
    {
        # Read the log file content and isolate the last run block
        $logContent = Get-Content -Path $logFile.FullName
        $lastBlockStart = ($logContent | Select-String -SimpleMatch -Pattern "Import-LnvPasswordChangeFile" | Select-Object -Last 1).LineNumber
        if ($lastBlockStart) { $logContent = $logContent[($lastBlockStart)..($logContent.Count - 1)] }

        # Define the expected success messages for each step
        $expectedSuccesses = @(
            "Targeting password type: Success",
            "Setting current password: Success",
            "Setting new password: Success",
            "Authenticating: Success",
            "Committing change: Success"
        )

        # Check if all expected success messages are present in the log
        $missing = @()
        foreach ($success in $expectedSuccesses)
        {
            if (-not ($logContent | Select-String -SimpleMatch -Pattern $success))
            {
                $missing += $success
            }
        }

        if ($missing.Count -eq 0)
        {
            Write-Output "Supervisor Password change detected as successful."
            exit 0  # Detection successful
        }
        else
        {
            Write-Output "Supervisor Password change not successful. Missing expected entries:"
            $missing | ForEach-Object { Write-Output " - $_" }
            exit 1  # Not detected
        }
    }
    else
    {
        Write-Output "Log file not found."
        exit 1  # Not detected
    }
    ```

### Assignment

No dependencies or supersedence required — assign the app to a device group to complete the deployment. Ideally, in an Autopilot pre-provisioned deployment, this would be added as a blocking app during ESP.

## Results

When complete, you should have 2 logs under **C:\ProgramData\Lenovo\ThinkBiosConfig\Logs**. Example output from each:

- PasswordChange.log

```dos
2026-04-20 11:12:46 [INFO]: === LENOVO PASSWORD CHANGE CONFIG INSTALL STARTED ===
2026-04-20 11:12:48 [SUCCESS]: Loaded module: 1.0.3
2026-04-20 11:12:48 [INFO]: Found password change file: C:\Windows\IMECache\32c3882a-fc0e-4ea5-becd-719f5376fe08_1\svp.ini
2026-04-20 11:13:04 [SUCCESS]: Password change configuration imported successfully.
2026-04-20 11:13:04 [SUCCESS]: === PASSWORD CHANGE CONFIG INSTALL COMPLETE (Exit 3010) ===
```

- TBC_Log.log (TBCT v2 generated)

```dos
Timestamp             Username       Message
2026-04-20 09:40:08 - <ComputerName> - Import-LnvPasswordChangeFile
2026-04-20 09:40:10 - <ComputerName> - Targeting password type: Success
Setting current password: Success
Setting new password: Success
Authenticating: Success
Committing change: Success
```

The detection script will make sure each line from the TBC_Log matches a **Success** to confirm the password change. If there's an **Access Denied** or **Invalid parameter** anywhere, the status details for the Win32 app will show not detected in Intune.

## Final Notes

- **Reboots:** A supervisor password change requires a reboot to take effect. The install command exits with code `3010`, which Intune will interpret as a soft reboot. Ensure **Device restart behavior** is set accordingly.
- **Retry limit:** If the wrong passphrase/secret key is supplied and the device fails consecutive install attempts, the BIOS may display a "Security password retry count exceeded" prompt on the next reboot.
- **Testing:** Validate against a single device in a lab environment before broad deployment.
