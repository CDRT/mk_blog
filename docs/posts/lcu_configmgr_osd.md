---
date:
    created: 2026-04-09
authors:
    - Phil
categories:
    - "2026"
title: Updating Lenovo BIOS, Drivers, and Firmware in a Configuration Manager Task Sequence Using the Lenovo Client Update Module
---

Installing current drivers, BIOS, and firmware during a bare metal OSD with the Lenovo Client Update PowerShell Module (LCU)
<!-- more -->

## Overview

This guide provides a step-by-step into integrating the new "Lenovo Client Update" [(LCU)](https://docs.lenovocdrt.com/guides/lcu/) PowerShell module into a Microsoft Configuration Manager (CM) Operating System Deployment (OSD) task sequence. The module automates the download and installation of the latest BIOS, drivers, and firmware for Lenovo devices during deployment, ensuring devices are up-to-date without manual intervention.

This approach assumes you're working in a CM environment with OSD task sequences. We'll cover preparing the module, creating a CM package, and adding a dedicated "Run PowerShell Script" step to your task sequence.

!!! note
    The module requires internet access during execution to download updates from Lenovo's servers. Test in a lab environment before production use.

## Prerequisites

- The Lenovo Client Update PowerShell module downloaded from the official Lenovo repository (e.g., via GitHub or Lenovo's scripting toolkit). Extract the module files, including `Lenovo.Client.Update.psd1` and any supporting files (e.g., `.psm1`).
- A source file share or directory accessible to CM for package creation.

## Step 1: Prepare the Lenovo Client Update Module and Script

You can install the module from the PowerShell gallery using this command during the task sequence:

```powershell
Install-Module -Name Lenovo.Client.Update -Scope CurrentUser
```

This demonstration will use the below steps to save/import the module during the task sequence to perform the necessary actions.

First, organize the module files and create the installation script in a source directory that will be used for the CM package.

1. Create a source directory (e.g., `\\Server\Sources\PowerShell\Modules\Lenovo`).
2. Use the **Save-Module** cmdlet to download the module to the source directory.

```powershell
Save-Module -Name Lenovo.Client.Update -Path "\\Server\Sources\PowerShell\Modules\Lenovo"
```

3. Create a PowerShell script file named `Install-LenovoUpdates.ps1` in the root of the source directory with the below sample code. This script imports the module, builds an Update Retriever style local repository on the device, and installs the latest drivers/BIOS/firmware.

   ```powershell
    # Install-LenovoUpdates.ps1
    [CmdletBinding()]
    param (
        [Parameter(
            HelpMessage = "Repository path containing Lenovo update packages"
        )]
        [string] $RepositoryPath = ""
    )

    # Import LCU from the current script directory
    try
    {
        $modulePath = Join-Path $PSScriptRoot "Lenovo.Client.Update.psd1"
        if (-not (Test-Path $modulePath))
        {
            throw "Lenovo.Client.Update.psd1 not found in $PSScriptRoot"
        }
        Import-Module $modulePath -Force -Verbose
        Write-Output "LCU module imported successfully"
    }
    catch
    {
        Write-Error $_.Exception.Message
        exit 1
    }

    # Retrieve all updates
    Get-LnvUpdatesRepo -RepositoryPath $RepositoryPath -RebootTypes '0,3,5'

    # Install all updates found
    Get-LnvUpdate -Repository $RepositoryPath -ScratchDirectory $RepositoryPath | Install-LnvUpdate -Path $RepositoryPath -ExportToWMI -Verbose

    # Wait for a few seconds to ensure all processes are settled
    Start-Sleep -Seconds 10

    # Re-check for any remaining updates after initial installation
    Get-LnvUpdate -Repository $RepositoryPath -ScratchDirectory $RepositoryPath | Install-LnvUpdate -Path $RepositoryPath -ExportToWMI -Verbose

    # Exit cleanly (0 = success)
    exit 0
   ```

   **Directory Structure**
   ```
   Lenovo.Client.Update\
    <ModuleVersion>\
        ├─ private
        ├─ public
        ├─ Install-LenovoUpdates.ps1
        ├─ Lenovo.Client.Update.Format.ps1xml
        ├─ Lenovo.Client.Update.psd1
        ├─ Lenovo.Client.Update.psm1
   ```

## Step 2: Create the Configuration Manager Package

Create a standard CM package (not an application) to host the module and script files.

1. In the CM console, navigate to **Software Library** > **Application Management** > **Packages**.
2. Right-click **Packages** and select **Create Package**.
3. Enter details:
    - **Name:** PSModule - Lenovo Client Update
    - **Description:** Package for LCU PowerShell Module
    - **Version**: `<ModuleVersion>`
    - **Source folder:** UNC path to your source directory (e.g., `\\Server\Sources\PowerShell\Modules\Lenovo\Lenovo.Client.Update\1.0.0`)
    - Check **This package contains source files**.
4. Do not create a program (this package is for content only).
5. Complete the wizard.
6. Distribute the package to your distribution points:
    - Right-click the package > **Distribute Content**.
    - Select relevant DPs and complete the process.

## Step 3: Add the Run PowerShell Script Step to the OSD Task Sequence

Integrate the package into your OSD task sequence by adding a dedicated step to run the script.

1. In the CM console, navigate to **Software Library** > **Operating Systems** > **Task Sequences**.
2. Edit your target OSD task sequence (e.g., right-click > **Edit**).
3. Choose an appropriate location for the step:
    - Recommended: After **Setup Windows and Configuration Manager**.
4. Add the step:
    - Click **Add** > **General** > **Run PowerShell Script**.
    - **Name:** LCU - Drivers/BIOS/Firmware
    - Select **This package contains the PowerShell script**.
    - Browse and select the package created in Step 2.
    - **Script name:** Install-LenovoUpdates.ps1
    - **Parameters** -RepositoryPath %_SMSTSMDataPath%\LenovoUpdates
    - **Execution policy:** Bypass..
5. Optional: Add conditions or options:
    - **Continue on error:** Enable if you want the TS to proceed even if some updates fail.
    - **Success codes:** Default (0, 3010 for reboot).
6. Apply changes and deploy the task sequence.

!!! note
    The repository will be built where the task sequence stores temporary cache files, resolving to C:`\`_SMSTaskSequence. This will be cleaned up after OSD completes. You can change the repository location to a different path if desired.

![](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image1.jpg)

## Client Side Experience

Once the process completes, reference the **InstallHistory.json** located under **%ProgramData\Lenovo\Lenovo.Client.Update\History** for installation status on each update.

![](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image2.jpg)

The **smsts.log** will also track details for each update as they're processed, including a signature check.

![](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image3.jpg)

## Final Notes

- **Reboots:** BIOS/firmware updates require reboots; ensure you have a **Restart Computer** added.
- **Testing:** Run the TS on a physical Lenovo device to validate.