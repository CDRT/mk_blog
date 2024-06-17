---
date:
    created: 2020-11-20
authors:
    - Phil
categories:
    - "2020"
title: Deploying Commercial Vantage with Intune
---
For those who are migrating from Configuration Manager to Intune, you may find the need to deploy Commercial Vantage to your devices from Intune. Commercial Vantage consists of a UWP app plus other services and components. An enterprise package with deployment guide and ADMX templates is available to assist deploying this solution. Download the latest version of Commercial Vantage with Deployment Guide [here.](https://support.lenovo.com/solutions/hf003321)
<!-- more -->
## Preparing the Win32 App

Once the zip has been downloaded and extracted, use the Content Prep [Tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool) to convert to an .intunewin format. There is already a provided batch file that handles the installation of all dependencies, certs, and .msix bundle so this will be used as the setup file. A sample command would be:

```cmd
IntuneWinAppUtil.exe -c "C:\IntuneWin\LenovoCommercialVantage_10.2010.11.0_v1" -s "setup-commercial-vantage.bat" -o "C:\IntuneWin\output" -q
```

![Create package](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image1.jpg)

## Creating the Win32 App

Login to the [MEM admin center](https://endpoint.microsoft.com/#blade/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/windowsApps) and add a new Windows app (Win32). Select the new App package file created above, which should be named **setup-commercial-vantage.intunewin** and click OK.

Fill out the necessary fields in the App information section and click Review + save

![Application information](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image2.jpg)

In the Edit application section, this is where the install/uninstall commands will be specified.

- Install command

```cmd
setup-commercial-vantage.bat
```

- Uninstall command

```cmd
C:\Windows\Sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File .\uninstall_vantage_v8\uninstall_all.ps1
```

Set Device restart behavior to **Determine behavior based on return codes**.

![Program details](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image3.jpg)

In the Requirements section, set the Operating system architecture to **64-bit** and Minimum operating system to **1809**

Add an additional Registry type requirement rule that will only apply to Lenovo branded systems.

![Requirement](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image4.jpg)

- Key path

```ini
HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS
```

- Value name

```ini
SystemManufacturer
```

- Registry key requirement: **String comparison**

- Operator: **Equals**

- Value

```ini
LENOVO
```

![Requirement rule](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image5.jpg)

For the detection rule, a custom script detection will be used. Commercial Vantage depends on these 2 services to run

- **ImControllerService** (Not required for ARM-based devices)
- **LenovoVantageService**

This sample PowerShell script can be used for detection

!!! info
    If the Store is not blocked in your environment, Vantage will automatically update itself as new versions are released.

```powershell
# Change this variable to the version you're deploying
$DeployedVantageVersion = "10.2305.30.0"

$ErrorActionPreference = "Stop"

try {

    If (Get-Service -Name ImControllerService) {
    
    }
        
    If (Get-Service -Name LenovoVantageService) {
        # Check for older of version of Vantage Service that causes UAC prompt. This is due to an expired certificate.  
        $minVersion = "3.8.23.0"
        $path = Join-Path -Path ${env:ProgramFiles(x86)} -ChildPath "Lenovo\VantageService\*\LenovoVantageService.exe" -Resolve
        $path = ${env:ProgramFiles(x86)} + "\Lenovo\VantageService\*\LenovoVantageService.exe"
        $version = (Get-ChildItem -Path $path).VersionInfo.FileVersion
        if ($version.Count -gt 1) {
            $version = $version[-1]
        }
            
        if ([version]$version -le [version]$minVersion) {
            
            Write-Output "Vantage Service outdated."; exit 1
        }
    }
        
    # Assume no version is installed
    $InstalledVersion = $false
    
    # For specific Appx version
    $InstalledVantageVersion = (Get-AppxPackage -Name E046963F.LenovoSettingsforEnterprise -AllUsers).Version

    If ([version]$InstalledVantageVersion -ge [version]$DeployedVantageVersion) {
        $InstalledVersion = $true
        
        # For package name only    
        # If (Get-AppxPackage -Name E046963F.LenovoSettingsforEnterprise -AllUsers) {

    }

    if ($InstalledVersion) {
        Write-Output "All Vantage Services and Appx Present"; exit 0
    }
    else {
        Write-Output "Commercial Vantage is outdated."; exit 1
    }
}
catch {
    
    Write-Output $_.Exception.Message; exit 1
}
```

![Detection rules](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image6.jpg)

Click Review and then Save to complete the app creation and content upload to Intune. Once the upload has finished, assign to a group.

## Automatically Create the Win32 App

A PowerShell helper script can be used to automatically create the Win32 app using the **IntuneWin32App** [module](https://www.powershellgallery.com/packages/IntuneWin32App).

Download the **New-CommercialVantageWin32.ps1** and **Detect-CommercialVantage.ps1** from my GitHub [here](https://github.com/philjorgensen/Intune/tree/main/Win32%20Apps/Commercial%20Vantage).

## Results

Track the installation through the **IntuneManagementExtension.log**

Here we can see the minimum OS version requirement has been met

![Log](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image7.jpg)

The additional requirement to check if the system is in fact a Lenovo system is true

![Log](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image8.jpg)

The PowerShell detection script finds both services and app are now present

![Log](https://cdrt.github.io/mk_blog/img/2020/cv_intune_deploy/image9.jpg)
