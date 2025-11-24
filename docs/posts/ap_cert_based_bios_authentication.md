---
date: 2025-11-24
authors:
    - Phil
categories:
    - "2025"
title: Deep Dive - Certificate-based Authentication in an Autopilot Pre-provisioning Deployment
---

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image1.png)

A deep dive on converting to certificate-based authentication in an Autopilot pre-provisioned deployment using Think BIOS Config and Certificates V2 tools
<!-- more -->

## Overview

This deep-dive guide walks you through **completely replacing a legacy BIOS Supervisor Password (SVP) with certificate-based authentication** on Lenovo Think devices—enabling secure, passwordless BIOS management at scale.

This solution is especially powerful in **Windows Autopilot pre-provisioned deployments**, where devices ship directly from the factory with an SVP set.

We'll cover:

- Generating an HSM-protected self-signed certificate in Azure Key Vault
- Installing the public certificate to devices in an Autopilot pre provisioning deployment
- Applying a configuration file to set desired BIOS settings using signed WMI commands

### Prerequisites

!!! warning "Before You Begin"

You must have:

- A commercial Lenovo Think device (ThinkPad/ThinkCentre/ThinkStation) with a **Supervisor Password already set** (ideally factory-set before drop-ship)
- Az.KeyVault and Az.Accounts PowerShell modules
- **Azure Key Vault – Premium tier** (required for HSM-backed keys when generating certificates)
- A service principal with the following **Azure RBAC roles scoped to the Key Vault**:
    - **Key Vault Administrator** – to create and manage certificates
      - **Key Vault Crypto User** – for signing commands with the private key


- [Lenovo BIOS Certificates Tool (LBCT)](https://docs.lenovocdrt.com/guides/lbct/)
- [Think BIOS Config Tool V2](https://docs.lenovocdrt.com/guides/tbct_v2/tbct_v2_top/)

## Step 1: Generate a Self-Signed Certificate in Azure Key Vault

We’ll create a self-signed X.509 certificate **directly inside Key Vault**.

> Only the **public certificate** will ever leave Key Vault.

Before running the script, update the variables with your values:

| Variable | What to put |
|---|---|
| `$AppId` | Application (client) ID of your service principal |
| `$Secret` | Client secret |
| `$keyVaultName` | Key Vault Name |
| `$certificateName` | Desired certificate name (must be unique in the vault) |
| Tenant ID in `Connect-AzAccount` | Your Entra ID tenant GUID (found in Entra ID → Overview) |

??? example "Create Azure Key Vault Certificate"
    ```powershell
    # Define variables
    $AppId = "00000000-0000-0000-0000-000000000000"
    $Secret = ConvertTo-SecureString "your-client-secret-here" -AsPlainText -Force
    $Credential = New-Object System.Management.Automation.PSCredential($AppId, $Secret)

    $keyVaultName = "keyvault-name"
    $Name = "certificate-name"

    # Replace with your actual tenant ID
    Connect-AzAccount -ServicePrincipal -Tenant "tenant-guid" -Credential $Credential

    # Certificate policy – customize as needed
    $policySplat = @{
        SubjectName       = "CN=Contoso BIOS Root CA 2025, O=Contoso Corp, C=US"
        IssuerName        = "Self"
        ValidityInMonths  = 12
        SecretContentType = "application/x-pem-file"
        KeyType           = "RSA"
        KeySize           = 2048
        ReuseKeyOnRenewal = $true
        KeyUsage          = @('digitalSignature', 'dataEncipherment')
    }

    $certificatePolicy = New-AzKeyVaultCertificatePolicy @policySplat

    Add-AzKeyVaultCertificate -VaultName $keyVaultName -Name $Name -CertificatePolicy $certificatePolicy

    Write-Host "Certificate creation triggered. Waiting 10 seconds for completion..."
    Start-Sleep -Seconds 10

    # Retrieve the certificate from Key Vault
    $cert = Get-AzKeyVaultCertificate -VaultName $keyVaultName -Name $Name

    # Export only the public cert as PEM
    $pemBytes = $cert.Certificate.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Cert)
    $pemBase64 = [Convert]::ToBase64String($pemBytes)

    $pemFormatted = @"
    -----BEGIN CERTIFICATE-----
    $pemBase64
    -----END CERTIFICATE-----
    "@

    # Save to temp (or copy this output manually)
    $pemPath = "$env:TEMP\$certificateName.pem"
    $pemFormatted | Set-Content -Path $pemPath
    Write-Host "Certificate saved to: $pemPath"
    ```

<screenshot of terminal showing certificate output>

## Step 2: Preparing the Configuration File

Prepare a source directory to hold all of the required content.

`C:\Sources\IntuneWin\ThinkBiosConfig`

Use the **Save-Module** command to download the **Lenovo.BIOS.Certificates** module here.

```powershell
Save-Module -Name Lenovo.BIOS.Certificates -Path C:\Sources\IntuneWin\ThinkBiosConfig
```

For this guide, I'll pick a few simple BIOS settings that will be configured on the devices.

!!! info "Module Reference"
    [https://docs.lenovocdrt.com/guides/tbct_v2/tbc_module_reference/]()

On an admin machine where the **Lenovo.BIOS.Config** module is installed/imported, open an elevated terminal and run the command:

```powershell
Initialize-LnvThinkBiosConfig
```

This will gather the WMI data needed for the module to work. No output is shown.

Then run the command:

```powershell
Show-LnvBiosSettings
```

You'll be presented with all of the available BIOS settings exposed through WMI

<screenshot of Show-LnvWmiSettings>

We're going to choose:

| settingName | defaultValue | settingOptions
|---|---| --- |
| WakeOnLANDock | `Enable` | {Disable, Enable}
| Allow3rdPartyUEFICA | `Disable` | {Disable, Enable}

To retrieve the options for a single setting, you can run:

```powershell
Get-LnvWmiSetting -Name Allow3rdPartyUEFICA
```

Let's flip these values

```powershell
Set-LnvWmiSetting -Name WakeOnLANDock -Value Disable

Set-LnvWmiSetting -Name Allow3rdPartyUEFICA -Value Enable
```

To confirm only the settings you've changed, run

```powershell
Show-LnvWmiSettings -OnlyChanged
```

Export these settings to a configuration file

```powershell
Export-LnvWmiSettings `
-ConfigFile "C:\Sources\IntuneWin\ThinkBiosConfig\thinkpadconfig.ini" `
-NoKey
```

We need to convert the configuration file into signed WMI commands using the Azure Key Vault key.

Assuming we’re still authenticated to Azure in the same session as when we created our certificate, set the following variables to pass to the -VaultName and -KeyName parameters.

```powershell
$configFile = "C:\Sources\IntuneWin\ThinkBiosConfig\thinkpadconfig.ini"

$keyVault = Get-AzKeyVault
$keyVaultName = $keyVault.Name
$signingKey = Get-AzKeyVaultKey -VaultName $keyVaultName -Name $Name

Convert-LnvBiosConfigFile `
-ConfigFile $configFile `
-VaultName $keyVaultName `
-KeyName $signingKey.Name `
-OutFile $configFile
```

Upon success, each BIOS setting in the configuration file is now converted into signed WMI commands to set and save the new settings.

Your source directory should now look like this

- C:\Sources\IntuneWin\ThinkBiosConfig
    - Lenovo.BIOS.Certificates
    - thinkpadconfig.ini

## Step 3: Create the First Win32 App in Intune – Convert to Certificate Authentication

Now that we have the exported certificate, it's time to prepare the first Win32 app. This will replace the Supervisor Password with the newly created certificate.

Copy your certificate PEM file from step 1 to the **ThinkBiosConfig** source directory.

Save the following script as a **Install-LnvBiosCertificate.ps1** here as well. This will provision the device with the certificate.

??? example "Install-LnvBiosCertificate.ps1"
    ```powershell
    # Install BIOS Certificate (Dependency App)
    # Exit: 1 = failed (triggers main app retry), 1641 = success + reboot

    $ErrorActionPreference = 'Stop'

    # === Log File ===
    $LogFile = "$env:ProgramData\Lenovo\BIOSCertificates\Logs\BIOSCertInstall.log"
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
    Write-Log "=== LENOVO BIOS CERTIFICATE INSTALL STARTED ===" 'INFO'

    # --- Paths ---
    $ModulePath = Join-Path $PSScriptRoot 'Lenovo.BIOS.Certificates\*\*.psd1'
    $CertFilePath = Join-Path $PSScriptRoot '.\*.pem'

    # --- Certificate Password ---
    $CertPassword = 'temppassword'  # Replace with temporary BIOS password set at factory

    # === Registry Tattoo Settings ===
    $RegPath = 'HKLM:\SOFTWARE\Lenovo\BIOSConfigDeployment'
    $RegCertStatus = 'CertStatus'
    $RegLastCert = 'LastCertInstall'
    $RegCertName = 'LastCertFile'

    # -------------------------------------------------
    # 1. Load Lenovo Certificates Module
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
        Write-Log "Failed to load BIOS module: $($_.Exception.Message)" 'ERROR'
        try
        {
            if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
            Set-ItemProperty -Path $RegPath -Name $RegCertStatus -Value 'Failed' -Type String -Force
            Set-ItemProperty -Path $RegPath -Name 'LastError' -Value "Module load failed: $($_.Exception.Message)" -Type String -Force
        }
        catch { Write-Log "Could not write failure tattoo" 'WARN' }
        exit 1
    }

    # -------------------------------------------------
    # 2. Verify Certificate File
    # -------------------------------------------------
    if (-not (Test-Path $CertFilePath -PathType Leaf))
    {
        $msg = "Certificate file not found: $CertFilePath"
        Write-Log $msg 'ERROR'
        try
        {
            if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
            Set-ItemProperty -Path $RegPath -Name $RegCertStatus -Value 'Failed' -Type String -Force
            Set-ItemProperty -Path $RegPath -Name 'LastError' -Value $msg -Type String -Force
        }
        catch { Write-Log "Could not write failure tattoo." 'WARN' }
        exit 1
    }
    Write-Log "Found certificate: $CertFilePath" 'INFO'

    # -------------------------------------------------
    # 3. Install Certificate
    # -------------------------------------------------
    try
    {
        $Result = Set-LnvBiosCertificate -CertFile $CertFilePath -Pass $CertPassword
        Write-Log "Certificate installation command executed." 'INFO'

        if ($null -eq $Result)
        {
            $msg = "No result returned from Set-LnvBiosCertificate"
            Write-Log $msg 'ERROR'
            throw $msg
        }

        $returnVal = $Result.ReturnValue

        Write-Log "Result: ReturnValue='$returnVal' | $env:COMPUTERNAME'" 'INFO'

        if ($returnVal -eq 'Success')
        {
            Write-Log "BIOS certificate installed successfully." 'SUCCESS'

            # === Tattoo Success ===
            try
            {
                if (-not (Test-Path $RegPath))
                {
                    New-Item -Path $RegPath -Force | Out-Null
                    Write-Log "Created registry path: $RegPath" 'INFO'
                }
                $certName = (Get-Item $CertFilePath).Name
                Set-ItemProperty -Path $RegPath -Name $RegCertStatus -Value 'Installed' -Type String -Force
                Set-ItemProperty -Path $RegPath -Name $RegLastCert -Value (Get-Date).ToString('yyyy-MM-dd HH:mm:ss') -Type String -Force
                Set-ItemProperty -Path $RegPath -Name $RegCertName -Value $certName -Type String -Force
                Write-Log "Registry tattoo: CertStatus='Installed', LastCertFile='$certName'" 'SUCCESS'
            }
            catch
            {
                Write-Log "Failed to write success tattoo: $($_.Exception.Message)" 'WARN'
            }

            Write-Log "=== CERTIFICATE INSTALL COMPLETE (Exit 1641) ===" 'SUCCESS'
            # Only on first install, hard reboot is required to convert to certificate-based auth.
            exit 1641
        }
        else
        {
            $msg = "CERTIFICATE INSTALL FAILED. ReturnValue='$returnVal'"
            Write-Log $msg 'ERROR'

            # === Tattoo Failure ===
            try
            {
                if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
                Set-ItemProperty -Path $RegPath -Name $RegCertStatus -Value 'Failed' -Type String -Force
                Set-ItemProperty -Path $RegPath -Name 'LastError' -Value $msg -Type String -Force
            }
            catch { Write-Log "Could not write failure tattoo" 'WARN' }

            exit 1
        }
    }
    catch
    {
        Write-Log "EXCEPTION during certificate install: $($_.Exception.Message)" 'ERROR'

        # === Tattoo Exception ===
        try
        {
            if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
            Set-ItemProperty -Path $RegPath -Name $RegCertStatus -Value 'Failed' -Type String -Force
            Set-ItemProperty -Path $RegPath -Name 'LastError' -Value "Exception: $($_.Exception.Message)" -Type String -Force
        }
        catch { Write-Log "Could not write exception tattoo" 'WARN' }

        exit 1
    }
    ```

Your source directory should now look this

- C:\Sources\IntuneWin\ThinkBiosConfig
    - Lenovo.BIOS.Certificates
    - Install-LnvBiosCertificate.ps1
    - thinkpadconfig.ini
    - publicCert.pem

Download the latest **Microsoft Win32 Content Prep Tool** (`IntuneWinAppUtil.exe`) from [GitHub](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool).

Open an elevated terminal and run:

```cmd
IntuneWinAppUtil.exe -c "C:\Sources\IntuneWin\ThinkBiosConfig -s "Install-LnvBiosCertificate.ps1" -o "C:\Sources\IntuneWin\Output" -q
```

This generates `Install-LnvBiosCertificate.intunewin`.

Navigate to **Microsoft Intune admin center → Apps → Windows → Create+ → Windows app (Win32)**.

### App information

- **Name:** `Convert to Cert-based Authentication`
- **Description:** `Replaces BIOS Supervisor Password with certificate-based authentication – runs once during Autopilot`
1. **Publisher:** `Lenovo / Your Org`

### Program
| Setting | Value |
|---|---|
| Install command | `%windir%\sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File ".\Install-LnvBiosCertificate.ps1"` |
| Uninstall command | `cmd.exe /c` |
| Install behavior | **System** |
| Device restart behavior | **Determine behavior based on return codes** |

### Requirements
| Setting | Value |
|---|---|
| Minimum operating system | Windows 11 21H2 |

> Additional requirement rule – Run only during OOBE/Autopilot

- **Rule type:** Script
- **Script:** (save/add the script below. Slightly modified from [here](https://oofhours.com/2023/09/15/detecting-when-you-are-in-oobe/))
- **Run script as 32-bit process on 64-bit clients:** No
- **Return type:** Boolean
- **Operator:** Equals → **No**

??? example "Requirement Rule Script"

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

### Detection rules
- **Rule type:** Registry
- **Key path:** `HKEY_LOCAL_MACHINE\SOFTWARE\Lenovo\BIOSConfigDeployment`
- **Value name:** `CertStatus`
- **Detection method:** String comparison
- **Operator:** Equals
- **Value:** `Installed`
- **Associated with a 32-bit app on 64-bit clients:** No

### Assignments
This will be a dependency app so you don't have to assign to a group.

### What Happens on the Device

1. The requirement script checks the device is in OOBE.
2. The script installs the public certificate on the device, replacing the Supervisor Password.
3. On success it exits with **1641** → device reboots immediately.
4. After reboot, the BIOS converts to **certificate-based authentication**.
5. Registry detection rule confirms the certificate status as installed → does not run again.

The device's BIOS is now secured with a certificate, ready for the second Win32 app that will apply signed WMI commands to configure BIOS settings using the private key stored in Azure Key Vault.

## Step 4: Create the Second Win32 App - Applying the Settings

Save the below script as **Set-LnvBiosSettings.ps1** to the **ThinkBiosConfig** source directory.

??? example "Set-LnvBiosSettings.ps1"

    ```powershell
    $ErrorActionPreference = 'Stop'

    # === Log File ===
    $LogFile = "$env:ProgramData\Lenovo\BIOSCertificates\Logs\BIOSConfig.log"
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
    Write-Log "=== BIOS Configuration Started ===" 'INFO'

    # --- Paths ---
    $ModulePath = Join-Path -Path $PSScriptRoot -ChildPath 'Lenovo.BIOS.Certificates\*\*.psd1' -Resolve
    $ConfigFilePath = Join-Path -Path $PSScriptRoot -ChildPath '*.ini' -Resolve

    # === Registry Tattoo Settings ===
    $RegPath = 'HKLM:\SOFTWARE\Lenovo\BIOSConfigDeployment'
    $RegNameConfig = 'LastAppliedConfig'
    $RegNameSuccess = 'LastAppliedSuccess'

    # -------------------------------------------------
    # 1. Hybrid Detection: Registry + WMI Fallback
    # -------------------------------------------------
    $CertInstalled = $false

    # --- Step 1: Check Registry Tattoo ---
    if (Test-Path $RegPath)
    {
        try
        {
            $regProps = Get-ItemProperty -Path $RegPath

            # Debug: Log all properties
            $regProps | Get-Member -MemberType NoteProperty | ForEach-Object {
                Write-Log "Registry entry: $($_.Name) = '$($regProps.($_.Name))'" 'INFO'
            }

            if ($regProps.CertStatus -eq 'Installed')
            {
                Write-Log "Registry confirms certificate installed (CertStatus=Success)" 'SUCCESS'
                $CertInstalled = $true
            }
            else
            {
                Write-Log "Registry exists but CertStatus is not Success (found: '$($regProps.CertStatus)')" 'WARN'
            }
        }
        catch
        {
            Write-Log "Failed to read registry key $($RegPath): $($_.Exception.Message)" 'WARN'
        }
    }
    else
    {
        Write-Log "Registry path $RegPath does not exist yet." 'INFO'
    }

    # --- Step 2: WMI Fallback ---
    if (-not $CertInstalled)
    {
        try
        {
            $BiosPassword = Get-CimInstance -Namespace root\WMI -ClassName Lenovo_BiosPasswordSettings
            $PasswordState = $BiosPassword.PasswordState
            Write-Log "WMI PasswordState: $PasswordState" 'INFO'

            # Password State 128 indicates certificate-based authentication (https://docs.lenovocdrt.com/ref/bios/wmi/wmi_guide/#detecting-password-state)
            if ($PasswordState -eq 128)
            {
                Write-Log "WMI confirms certificate installed (PasswordState=128)" 'SUCCESS'
                $CertInstalled = $true

                # Sync registry
                try
                {
                    if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
                    Set-ItemProperty -Path $RegPath -Name 'CertStatus' -Value 'Success' -Type String -Force
                    Set-ItemProperty -Path $RegPath -Name 'LastCertInstall' -Value (Get-Date).ToString('yyyy-MM-dd HH:mm:ss') -Type String -Force
                    Write-Log "Updated registry tattoo from WMI detection" 'INFO'
                }
                catch
                {
                    Write-Log "Could not update registry from WMI: $($_.Exception.Message)" 'WARN'
                }
            }
        }
        catch
        {
            Write-Log "WMI query failed: $($_.Exception.Message)" 'WARN'
        }
    }

    # --- Step 3: Final Decision ---
    if (-not $CertInstalled)
    {
        $msg = "BIOS certificate not detected (registry or WMI). Waiting for dependency app."
        Write-Log $msg 'ERROR'
        try
        {
            if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
            Set-ItemProperty -Path $RegPath -Name 'Status' -Value 'Failed' -Type String -Force
            Set-ItemProperty -Path $RegPath -Name 'LastError' -Value $msg -Type String -Force
        }
        catch { }
        Write-Log "Exiting 1 - will retry after cert app" 'INFO'
        exit 1
    }

    Write-Log "Certificate confirmed. Proceeding with BIOS config." 'SUCCESS'

    # -------------------------------------------------
    # 2. Import Lenovo BIOS Module
    # -------------------------------------------------
    try
    {
        $Module = Import-Module -Name $ModulePath -PassThru -Force
        Write-Log "Successfully imported module: $($Module.Name)" 'SUCCESS'
    }
    catch
    {
        Write-Log "Failed to import Lenovo BIOS module from '$ModulePath': $($_.Exception.Message)" 'ERROR'
        exit 1
    }

    # -------------------------------------------------
    # 3. Verify Config File Exists
    # -------------------------------------------------
    if (-not (Test-Path -Path $ConfigFilePath -PathType Leaf))
    {
        Write-Log "Configuration file not found: '$ConfigFilePath'" 'ERROR'
        exit 1
    }
    Write-Log "Found config file: $ConfigFilePath" 'INFO'

    # -------------------------------------------------
    # 4. Submit BIOS Config & Validate ALL Results
    # -------------------------------------------------
    try
    {
        $Result = Submit-LnvBiosConfigFile -ConfigFile $ConfigFilePath
        Write-Log "BIOS configuration file submitted: '$ConfigFilePath'" 'INFO'
        Write-Log "Refer to BiosCerts.log under %ProgramData%\Lenovo\BIOSCertificates\Logs" 'INFO'

        if ($Result -and $Result.Count -gt 0)
        {
            $allSuccess = $true
            $failureDetails = @()
            $successCount = 0

            foreach ($setting in $Result)
            {
                $returnVal = if ($setting.ReturnValue) { $setting.ReturnValue } else { 'Unknown' }
                Write-Log "  -> Setting result Betting: ReturnValue='$returnVal' | $env:COMPUTERNAME" 'INFO'

                if ($returnVal -eq 'Success')
                {
                    $successCount++
                }
                else
                {
                    $allSuccess = $false
                    $failureDetails += "ReturnValue='$returnVal'"
                }
            }

            if ($allSuccess)
            {
                Write-Log "All $($Result.Count) settings applied (ReturnValue: Success)" 'SUCCESS'

                # === Tattoo Registry on Success ===
                try
                {
                    if (-not (Test-Path $RegPath))
                    {
                        New-Item -Path $RegPath -Force | Out-Null
                        Write-Log "Created registry path: $RegPath" 'INFO'
                    }
                    $configName = (Get-Item $ConfigFilePath).Name
                    Set-ItemProperty -Path $RegPath -Name $RegNameConfig -Value $configName -Type String -Force
                    Set-ItemProperty -Path $RegPath -Name $RegNameSuccess -Value (Get-Date).ToString('yyyy-MM-dd HH:mm:ss') -Type String -Force
                    Set-ItemProperty -Path $RegPath -Name 'Status' -Value 'Success' -Type String -Force
                    Write-Log "Registry tattoo applied: $RegNameConfig='$configName', Status='Success'" 'SUCCESS'
                }
                catch
                {
                    Write-Log "Failed to write success tattoo: $($_.Exception.Message)" 'WARN'
                }
            }
            else
            {
                $msg = "PARTIAL FAILURE: $successCount/$($Result.Count) succeeded. Failed: $($failureDetails -join '; ')"
                Write-Log $msg 'ERROR'
                try
                {
                    if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
                    Set-ItemProperty -Path $RegPath -Name 'Status' -Value 'Failed' -Type String -Force
                    Set-ItemProperty -Path $RegPath -Name 'LastError' -Value $msg -Type String -Force
                }
                catch { Write-Log "Failed to write failure tattoo" 'WARN' }
                Write-Log "Exiting with code 1 (partial failure)" 'INFO'
                exit 1
            }
        }
        else
        {
            $msg = "No results returned from Submit-LnvBiosConfigFile"
            Write-Log $msg 'ERROR'
            try
            {
                if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
                Set-ItemProperty -Path $RegPath -Name 'Status' -Value 'Failed' -Type String -Force
                Set-ItemProperty -Path $RegPath -Name 'LastError' -Value $msg -Type String -Force
            }
            catch { Write-Log "Failed to write failure tattoo" 'WARN' }
            exit 1
        }
    }
    catch
    {
        Write-Log "EXCEPTION in Submit-LnvBiosConfigFile: $($_.Exception.Message)" 'ERROR'
        try
        {
            if (-not (Test-Path $RegPath)) { New-Item -Path $RegPath -Force | Out-Null }
            Set-ItemProperty -Path $RegPath -Name 'Status' -Value 'Failed' -Type String -Force
            Set-ItemProperty -Path $RegPath -Name 'LastError' -Value $_.Exception.Message -Type String -Force
        }
        catch { Write-Log "Failed to write exception tattoo" 'WARN' }
        exit 1
    }

    # -------------------------------------------------
    # 5. Success → Request Reboot
    # -------------------------------------------------
    Write-Log "=== BIOS CONFIG COMPLETED SUCCESSFULLY ===" 'SUCCESS'
    Write-Log "Exit code: 1641 (reboot required to apply settings)" 'INFO'
    exit 1641
    ```

The source directory should look like this:

- C:\Sources\IntuneWin\ThinkBiosConfig
    - Lenovo.BIOS.Certificates
    - Install-LnvBiosCertificate.ps1
    - publicCert.pem
    - Set-LnvBiosSettings.ps1
    - thinkpadconfig.ini


### App information

- **Name:** `Set Lenovo BIOS Settings`
- **Description:** `Configure Lenovo BIOS settings using signed WMI commands`
- **Publisher:** `Lenovo / Your Org`

### Program
| Setting | Value |
|---|---|
| Install command | `%windir%\sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File ".\Set-LnvBiosSettings.ps1"` |
| Uninstall command | `cmd.exe /c` |
| Install behavior | **System** |
| Device restart behavior | **Determine behavior based on return codes** |

### Requirements
| Setting | Value |
|---|---|
| Minimum operating system | Windows 11 21H2 |

> Additional requirement rule – Run only during OOBE/Autopilot

- **Rule type:** Script
- **Script:** (same script as above)
- **Run script as 32-bit process on 64-bit clients:** No
- **Return type:** Boolean
- **Operator:** Equals → **No**

??? example "Requirement Rule Script"

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

### Detection rules
> Rule #1

- **Rule type:** Registry
- **Key path:** `HKEY_LOCAL_MACHINE\SOFTWARE\Lenovo\BIOSConfigDeployment`
- **Value name:** `CertStatus`
- **Detection method:** String comparison
- **Operator:** Equals
- **Value:** `Installed`
- **Associated with a 32-bit app on 64-bit clients:** No

> Rule #2

- **Rule type:** Registry
- **Key path:** `HKEY_LOCAL_MACHINE\SOFTWARE\Lenovo\BIOSConfigDeployment`
- **Value name:** `Status`
- **Detection method:** String comparison
- **Operator:** Equals
- **Value:** `Success`
- **Associated with a 32-bit app on 64-bit clients:** No

### Dependencies
Add the first Win32 app here and toggle the option to automatically install the app.

### Assignments
Assign to a device group containing Autopilot registered devices with an Autopilot deployment profile also assigned.

### What Happens on the Device
1. Pre-provisioned deployment starts
2. Dependency App #1 runs → installs public cert → converts BIOS to cert auth → reboots (1641)
3. After reboot → ESP continues → App #2 runs → confirms certificate is installed via registry + WMI → submits signed `.ini` → settings applied → reboots
4. All detection rules pass → ESP continues and reseals

## Final Notes
The new version of the Think BIOS Config tool is not capable of initially setting a Supervisor password. The tool only exposes settings and their values through WMI. It's still recommended to have a Supervisor password set at the factory.

You can combine different settings across supported Think-branded products into a single configuration file for simplicity sake.

In a future version of the tools, specifying the Supervisor password in plain text will be replaced with an encrypted string for security purposes.