---
date:
    created: 2019-10-29
    updated: 2025-07-24
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

!!!info
    In previous versions of Commercial Vantage, the installation was orchestrated using a batch file. Starting in version 20.2506.39.0, installation of the Commercial Vantage suite and add-ins are controlled by [**VantageInstaller.exe**](https://docs.lenovocdrt.com/guides/cv/commercial_vantage/#using-vantageinstallerexe)

## Create the App

Download/extract the contents from the zip to a source location.

In the console, navigate to **Software Library > Application Management > Applications**. Click Create Application and set the following:

* **General**: Manually specify the application information

    - **General Information**: Enter Commercial Vantage for the name and any other information you want to fill out.

* **Software Center**: Fill out what should be displayed to the end user when they view this app in Software Center.

    - **Deployment Types**: Add a **Script Installer** deployment type

    - **General Information**: Enter a name for the deployment type.

    - **Content**: Point the content location to the directory where the Vantage source files reside.

        - Installation Program: **VantageInstaller.exe Install -Vantage -SuHelper**

        - Uninstall Program: **VantageInstaller.exe Uninstall -Vantage**

!!! note
    Depending on your requirements, there can be two detection methods that can be used. If the Store is blocked in your environment and/or Store apps are not automatically updated, detection method 1 should be used. Otherwise, detection method 2 can be used as future versions will be auto updated.

* **Detection Method 1**: Select **Configure rules to detect the presence of this deployment type**. Click **Add Clause...**

    - Setting Type: **File System**

    - Type: **Folder**

    - Path: **ProgramFiles\Lenovo**

    - File or folder name: **VantageService**

    - Tick the box **This file or folder is associated with a 32-bit application on 64-bit systems**.

    - Ensure the radio button **The file system setting must exist on the target system to indicate presence of this application** is selected.

    - Add a second **File System** clause to check the presence of the .appx package

    - Type: **Folder**

    - Path: **ProgramFiles\WindowsApps**

    - File or folder name: **E046963F.LenovoSettingsforEnterprise_20.2506.39.0_x64__k1h2ywk1493x8**

    - Tick the box **This file or folder is associated with a 32-bit application on 64-bit systems**.

---

- **Detection Method 2**: Select **Use a custom script to detect the presence of this deployment type**.
    - Select **PowerShell** as the script type. Enter the following code into the script contents field:

The script can also be downloaded from my [GitHub](https://github.com/philjorgensen/ConfigMgr/blob/main/Applications/Detect-CommercialVantage.ps1)

```powershell title="Detect-CommercialVantage.ps1"
# Version of Lenovo Vantage that is being deployed
$DeployedVantageVersion = [version]"20.2506.39.0"

try
{
    # Get the path to the most recent VantageService folder under ProgramFiles(x86)
    $vantageServicePath = Get-ChildItem -Path "${env:ProgramFiles(x86)}\Lenovo\VantageService" -Directory | Select-Object -Last 1

    # Check if the path exists before proceeding
    if ($vantageServicePath)
    {
        # Find LenovoVantageService.exe in the directory
        $vantageServiceFile = Get-ChildItem -Path $vantageServicePath.FullName -Filter "LenovoVantageService.exe" -File -Recurse -ErrorAction Stop | Select-Object -Last 1

        if ($vantageServiceFile)
        {
            # Extract the version information
            $serviceVersion = [version]$vantageServiceFile.VersionInfo.FileVersion
        }
        else
        {
            $serviceVersion = $null
            Write-Warning "LenovoVantageService.exe was not found."
        }
    }
    else
    {
        $serviceVersion = $null
        Write-Warning "VantageService directory was not found."
    }


    $minServiceVersion = [version]"3.8.23.0"
    if ($serviceVersion -le $minServiceVersion)
    {
        Write-Output "Lenovo Vantage Service is outdated (found version $serviceVersion, required minimum $minServiceVersion)."
        exit 1
    }
}
catch
{
    Write-Output "Failed to retrieve Lenovo Vantage Service version. Error: $($_.Exception.Message)"
    exit 1
}

# Check for the Lenovo Commercial Vantage APPX package
try
{
    $vantagePackage = Get-AppxPackage -Name E046963F.LenovoSettingsforEnterprise -AllUsers -ErrorAction Stop
    $installedVersion = [version]$vantagePackage.Version

    if ($installedVersion -ge $DeployedVantageVersion)
    {
        Write-Output "Lenovo Commercial Vantage APPX package is up-to-date (installed version: $installedVersion, required version: $DeployedVantageVersion)."
        exit 0
    }
    else
    {
        Write-Output "Lenovo Commercial Vantage APPX package is outdated (installed version: $installedVersion, required version: $DeployedVantageVersion)."
        exit 1
    }
}
catch
{
    Write-Output "Failed to detect Lenovo Commercial Vantage APPX package. Error: $($_.Exception.Message)"
    exit 1
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
