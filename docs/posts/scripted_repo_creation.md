---
date: 
    created: 2023-03-13
    updated: 2024-11-19
authors:
    - Phil
    - Joe
categories: 
    - "2023"
title: Creating Local Repository Using PowerShell
cover_image: ../img/2023/scripted_repo_creation/ps_icon.png
---

There are various scenarios where one might want to quickly generate a local repository of Lenovo updates that can be consumed by Thin Installer or System Update in a scripted manner. This article will describe a PowerShell script that can be leveraged to create a repository for a specified machine type and OS. A scenario where this script might be used will also be described.
<!-- more -->
The script, **Get-LnvUpdatesRepo.ps1**, can be found in the CDRT Library repository on GitHub [here](https://github.com/CDRT/Library). The parameters and switches used by the script are documented at the beginning of the script.

The child task sequence described in the first scenario can be downloaded [here](https://download.lenovo.com/cdrt/eval/GetLenovoUpdates.zip).

## Scenario 1

In the first scenario, Thin Installer will be leveraged in a Microsoft Configuration Manager Operating System Deployment task sequence to apply any applicable updates available for the machine type of the targeted system. For this approach, a PowerShell script will be executed in a child task sequence to orchestrate the creation of the repository with the desired updates which can be customized based on Package Types and Reboot Types.

### Child Task Sequence Workflow

The child task sequence will be added after the **Setup Windows and Configuration Manager** step in your parent Operating System Deployment task sequence

#### Preparing Thin Installer

The top level group **Prepare Thin Installer** queries the device to determine if it is a ThinkPad, ThinkCentre or ThinkStation commercial PC product.
![Prepare Thin Installer](https://cdrt.github.io/mk_blog/img/2023/scripted_repo_creation/image1.jpg)

A WQL query is used to check if the Operating System build is 22H1 or earlier. If it is, then Thin Installer is installed as an Application in the 'Application' group:
![App: Thin Installer](https://cdrt.github.io/mk_blog/img/2023/scripted_repo_creation/image2.jpg)
![App: Thin Installer](https://cdrt.github.io/mk_blog/img/2023/scripted_repo_creation/image3.jpg)

Another WQL query is used to check if the Operating System build is 22H2 or later in the 'Winget' group. This group contains a step to install Thin Installer from the Winget repository. Windows 11 22H2 contains Winget automatically while earlier versions of Windows must install Winget from the Microsoft.DesktopAppInstaller package. If Winget is available in your environment on earlier OS builds, you can change the conditions on these groups and leverage the Winget install task instead.
![Winget: Thin Installer](https://cdrt.github.io/mk_blog/img/2023/scripted_repo_creation/image4.jpg)

PowerShell script to install Thin Installer using Winget:

```powershell
# Function to install Thin Installer
function Install-ThinInstaller
{
    $ErrorActionPreference = 'SilentlyContinue'
    
    # Create folder for logging
    $WinGet = "$($env:ProgramData)\WinGet"
    if (-not (Test-Path -Path $WinGet))
    {
        New-Item -ItemType Directory -Path $WinGet
    }

    # Select newest winget.exe file based on folder order and set it as winget variable
    $wingetExe = Get-ChildItem -Path "$($env:ProgramFiles)\WindowsApps" -Filter winget.exe -Recurse | Sort-Object -Property 'FullName' -Descending | Select-Object -First 1 -ExpandProperty FullName | Tee-Object -FilePath "$WinGet\Winget-file-found-from.log"

    Write-Output "Thin Installer not present. Installing the latest version from the Winget repository."
    Start-Process -FilePath $wingetExe -NoNewWindow -Wait -ArgumentList 'install Lenovo.ThinInstaller --silent --accept-package-agreements --accept-source-agreements'
    Start-Sleep -Seconds 15
}

# Function to check and update Thin Installer
function Update-ThinInstaller
{
    $ThinInstallerPath = Join-Path -Path (Join-Path -Path ${env:ProgramFiles(x86)} -ChildPath Lenovo) -ChildPath "ThinInstaller"
    
    if (-not (Test-Path -Path $ThinInstallerPath))
    {
        Install-ThinInstaller
    }
    else
    {
        Write-Output "Checking the Winget repository for an updated version..."

        # Get the local version of Thin Installer
        [version]$LocalVersion = $(try { ((Get-ChildItem -Path $ThinInstallerPath -Filter "thininstaller.exe" -Recurse).VersionInfo.FileVersion) } catch { $null })

        # Prepare for capturing the versions output
        $versionsOutputFile = "$($env:TEMP)\winget_versions_output.txt"

        # Select newest winget.exe file based on folder order and set it as winget variable
        $wingetExe = Get-ChildItem -Path "$($env:ProgramFiles)\WindowsApps" -Filter winget.exe -Recurse | Sort-Object -Property 'FullName' -Descending | Select-Object -First 1 -ExpandProperty FullName | Tee-Object -FilePath "$WinGet\Winget-file-found-from.log"
    
        # Start the process to get versions and redirect output to a file
        Start-Process -FilePath $wingetExe -ArgumentList 'show lenovo.thininstaller --versions' -NoNewWindow -Wait -RedirectStandardOutput $versionsOutputFile

        # Read the output from the file
        $versionsOutput = Get-Content -Path $versionsOutputFile

        # Extract versions from the output and find the latest version
        $versions = $versionsOutput | Select-String -Pattern '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$'
        $latestVersion = [version]($versions -replace 'Version', '' -replace '-', '' | ForEach-Object { $_.Trim() } | Sort-Object -Descending | Select-Object -First 1)

        Write-Output "Local version: $LocalVersion"
        Write-Output "Latest version: $latestVersion"

        # Compare the local version with the latest version
        if ($LocalVersion -lt $latestVersion)
        {
            Write-Output "An update is available."
            Install-ThinInstaller
        }
        else
        {
            Write-Output "The installed version is up-to-date."
        }
    }
}

# Call the function to check and update Thin Installer
Update-ThinInstaller
```

#### Building the Repository

The PowerShell script **Get-LnvUpdatesRepo.ps1** will download current updates from Lenovo's servers and store on the device. Parameters are accepted to specify the repository path and to filter updates by package type and reboot type. The [SCCM Task Sequence module](https://github.com/sombrerosheep/TaskSequenceModule/tree/master) has also been integrated to provide more details as updates are downloaded and installed during each pass of Thin Installer.

```cmd
-RepositoryPath 'C:\Program Files (x86)\Lenovo\ThinInstaller\Repository' -PackageTypes '1,2,3,4' -RebootTypes '0,3,5' -RT5toRT3
```

Based on the above, this filters update package types to include:

  - 1 - Applications
  - 2 - Drivers
  - 3 - BIOS
  - 4 - Firmware

and reboot packages that are:

  - 0 (No reboot required)
  - 3 (Reboot required but not forced)
  - 5 (Delayed forced reboot).

The **-RT5toRT3** parameter is a special case to be used in task sequences which will control the rebooting of the system after Thin Installer runs. It will change Reboot Type 5 updates to Reboot Type 3, which instructs Thin Installer to not perform the reboot.

!!! note
    After these types of updates are applied, the system needs to be rebooted immediately to avoid possible damage to the system. This is taken care of by the task sequence.

!!! info
    The **All Updates** group contains the necessary parameters and Thin Installer command lines to include Reboot Type 5 packages (BIOS/Firmware). The **Drivers** group will only download Reboot Type 0 and 3 packages (Drivers), and is disabled by default.

Once all content is downloaded to the device, 3 passes of Thin Installer (with a reboot in between) installs all updates to ensure the device is current.

#### Install Passes

A PowerShell script invoking Thin Installer with the necessary parameters is executed in each pass.

**1st Pass**

If a BIOS update is applicable, this package gets installed first

**2nd Pass**

Only drivers and apps are filtered for installation.

!!! note
    In some cases, typically with Thunderbolt, there may be a requirement that the latest driver needs to be installed *before* the firmware can be updated. This pass will ensure those drivers are installed before the firmware is installed in the next pass.

A preview of what the task sequence looks like during these passes

![TI-Install](https://cdrt.github.io/mk_blog/img/2023/scripted_repo_creation/image6.jpg)

![TI-Install](https://cdrt.github.io/mk_blog/img/2023/scripted_repo_creation/image7.jpg)

**3rd Pass**

The final pass installs firmware packages, such as Intel ME Firmware.

!!! note
    In some cases there are drivers that do not become applicable until another driver has been installed. This final pass includes drivers to ensure those are covered for a complete installation.

### Summary

With this scenario, there is very little effort needed to manage multiple models. This approach does download the updates on each device being deployed which may be a redundant use of network bandwidth to the Internet. However, this approach works very well in a scenario where the device is being reimaged off-site.

This approach could be modified easily to incorporate a centralized network storage for the repository so that multiple devices could share the same content. The **Get-LnvUpdatesRepo.ps1** script could be called from an admin's machine to create repository folders for each machine type. The script in the task sequence could then be modified to call Thin Installer with a -repository parameter pointing to the correct machine type folder.

Normally, if BIOS and firmware updates are included which will force a reboot (reboot type 1) or force a delayed reboot (reboot type 5) then the task sequence will be interrupted. Most current Lenovo ThinkPad, ThinkCentre and ThinkStation products have switched to using Reboot Type 5 for BIOS updates. Therefore, the **Get-LnvUpdatesRepo.ps1** script has an option, **-RT5toRT3**, which changes the Reboot Type to 3 instead of 5. This, in combination with the **-noreboot** parameter of Thin Installer, allows the updates to be applied with Configuration Manager controlling the restart.

In a future article we will look at other scenarios that are made possible with the **Get-LnvUpdatesRepo.ps1** script and may even provide additional features and enhancements to the script.
