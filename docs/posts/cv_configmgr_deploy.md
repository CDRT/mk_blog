---
date:
    created: 2019-10-29
    updated: 2023-02-12
authors:
    - Phil
categories:
    - "2019"
title: Deploying Commercial Vantage with ConfigMgr
---

Previously, Lenovo provided two separate apps (Lenovo Settings and Lenovo Companion) that allowed the user to change hardware settings, run diagnostic scans, and check for software and driver updates.  As of December 2017, all of the features in those two apps (discontinued) were merged into a single app [**Commercial Vantage**](https://support.lenovo.com/solutions/hf003321)

This post will walk through deploying Commercial Vantage as a ConfigMgr [application](https://docs.microsoft.com/mem/configmgr/apps/deploy-use/create-applications).
<!-- more -->
All required components, as well as the Group Policy Admin Template, and sample registry files are included in the zip available for download on the Vantage landing page.

## Create the App

Download/extract the contents from the zip to a source location.

In the console, navigate to **Software Library > Application Management > Applications**. Click Create Application and set the following:

* **General**: Manually specify the application information

    - **General Information**: Enter Commercial Vantage for the name and any other information you want to fill out.

* **Software Center**: Fill out what should be displayed to the end user when they view this app in Software Center.

    - **Deployment Types**: Add a **Script Installer** deployment type

    - **General Information**: Enter a name for the deployment type.  

    - **Content**: Point the content location to the directory where the Vantage source files reside.

        - Installation Program: **setup-commercial-vantage.bat**

        - Uninstall Program: **powershell.exe -ExecutionPolicy Bypass -File .\uninstall_vantage_v8\uninstall_all.ps1**

!!! note
    Depending on your requirements, there can be two detection methods that can be used. If the Store is blocked in your environment and/or Store apps are not automatically updated, detection method 1 should be used. Otherwise, detection method 2 can be used as future versions will be auto updated.

* **Detection Method 1**: Select **Configure rules to detect the presence of this deployment type**. Click **Add Clause...**

    - Setting Type: **File System**

    - Type: **Folder**

    - Path: **ProgramFiles\Lenovo**

    - File or folder name: **VantageService**

    !!! info ""
        This directory is what's created once the System Interface Foundation driver has been installed

    - Tick the box **This file or folder is associated with a 32-bit application on 64-bit systems**.

    - Ensure the radio button **The file system setting must exist on the target system to indicate presence of this application** is selected.

    - Add a second **File System** clause to check the presence of the .appx package

    - Type: **Folder**

    - Path: **ProgramFiles\WindowsApps**

    - File or folder name: **E046963F.LenovoSettingsforEnterprise_10.2110.11.0_x64__k1h2ywk1493x8**

    - Tick the box **This file or folder is associated with a 32-bit application on 64-bit systems**.

---

- **Detection Method 2**: Select **Use a custom script to detect the presence of this deployment type**.
    - Select **PowerShell** as the script type. Enter the following code into the script contents field:

The script can also be downloaded from my [GitHub](https://github.com/philjorgensen/ConfigMgr/blob/main/Applications/Detect-CommercialVantage.ps1)

```powershell title="Detect-CommercialVantage.ps1"
# Change this variable to the version you're deploying
$DeployedVantageVersion = "10.2310.28.0"

$ErrorActionPreference = "Stop"

try
{

    If (Get-Service -Name ImControllerService)
    {
    
    }
        
    If (Get-Service -Name LenovoVantageService)
    {
        # Check for older of version of Vantage Service that causes UAC prompt. This is due to an expired certificate.  
        $minVersion = "3.8.23.0"
        $path = ${env:ProgramFiles(x86)} + "\Lenovo\VantageService\*\LenovoVantageService.exe"
        $version = (Get-ChildItem -Path $path).VersionInfo.FileVersion
            
        if ([version]$version -le [version]$minVersion)
        {
            
            Write-Output "Vantage Service outdated."
        }
    }
        
    # Assume no version is installed
    $InstalledVersion = $false
    
    # For specific Appx version
    $InstalledVantageVersion = (Get-AppxPackage -Name E046963F.LenovoSettingsforEnterprise -AllUsers).Version

    If ([version]$InstalledVantageVersion -ge [version]$DeployedVantageVersion)
    {
        $InstalledVersion = $true
        
        # For package name only    
        # If (Get-AppxPackage -Name E046963F.LenovoSettingsforEnterprise -AllUsers) {

    }

    if ($InstalledVersion)
    {
        Write-Host "All Vantage Services and Appx Present"
    }
    else
    {
        # Write-Output "Commercial Vantage is outdated."
    }
}
catch
{
    
}
```

- User Experience:
    - Installation Behavior: **Install for system**

    - Logon requirement: **Whether or not a user is logged on**

    - Installation program visibility: **Hidden**

## Distribute Content

Select the **Commercial Vantage** application and click **Distribute Content** from the ribbon bar. Both apps should be shown in the **Content to distribute** list. Click next and add the Distribution Points or Distribution Point Group to send the content to.

## Deploy the App

Create a Device Collection to deploy the **Commercial Vantage** app. If you have a mixed environment of computer vendors, it's suggested to create a collection targeting only Lenovo Think branded products.
