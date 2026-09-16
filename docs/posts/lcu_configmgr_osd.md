---
date:
    created: 2026-04-09
    updated: 2026-09-16
authors:
    - Phil
categories:
    - "2026"
title: Updating Lenovo BIOS, Drivers, and Firmware in a Configuration Manager Task Sequence Using the Lenovo Client Update Module
---

Installing current drivers, BIOS, and firmware during a bare metal OSD with the Lenovo Client Update PowerShell Module (LCU)
<!-- more -->

!!! info "Updated September 2026"
    This post now leads with a new method: the LCU scripts pasted inline into a child task sequence, with BIOS and firmware installed first and drivers installed in passes. The [original packaged-script method](#original-method-packaged-script) follows it and still works.

## Overview

This guide provides a step-by-step into integrating the new "Lenovo Client Update" [(LCU)](https://docs.lenovocdrt.com/guides/lcu/) PowerShell module into a Microsoft Configuration Manager (CM) Operating System Deployment (OSD) task sequence. The module automates the download and installation of the latest BIOS, drivers, and firmware for Lenovo devices during deployment, ensuring devices are up-to-date without manual intervention.

This post covers two ways to run it:

| | Recommended method | Original method |
|---|---|---|
| Delivery | Scripts pasted inline into a child task sequence | Script and module in a CM package |
| Module source | PowerShell Gallery, downloaded during the task sequence | CM content on your distribution points |
| Changing a script | Edit the task sequence step | Update the source files and redistribute the package |
| Install order | BIOS/firmware, conditional restart, then drivers in passes | All update types in one step |
| Restart handling | Built in, driven by the `RebootMandatory` variable | Add and position your own **Restart Computer** step |

The recommended method suits most environments. The original method fits when devices cannot reach the PowerShell Gallery during imaging, or when you want the module version to change only when you update CM content.

!!! note
    The module requires internet access during execution to download updates from Lenovo's servers. Test in a lab environment before production use.

## Prerequisites

Both methods:

- A CM environment with an OSD task sequence that targets Lenovo commercial devices.
- Internet access from the device during the task sequence, to download update packages from Lenovo.

Recommended method:

- Access from the device to `www.powershellgallery.com` during the task sequence.

Original method:

- The Lenovo Client Update PowerShell module downloaded from the official Lenovo repository (e.g., via GitHub or Lenovo's scripting toolkit). Extract the module files, including `Lenovo.Client.Update.psd1` and any supporting files (e.g., `.psm1`).
- A source file share or directory accessible to CM for package creation.

## Recommended Method: Inline Scripts in a Child Task Sequence

This method pastes each script into its **Run PowerShell Script** step using the **Enter a PowerShell script** option, which opens an in-field editor. There is no CM package, source share, or content distribution to maintain, and changing a script is a task sequence edit.

All five scripts are available in my [ConfigMgr repository](https://github.com/philjorgensen/ConfigMgr/tree/main/OSD/Lenovo-Client-Update), along with a README covering the step layout, settings, and task sequence variables. The copies in this post leave out the comment-based help to keep them short for pasting. If the post and the repository ever differ, the repository has the current version.

Building this as a child task sequence keeps the update logic in one place. The parent OSD sequence calls it with a single **Run Task Sequence** step, and the same child can be called from a maintenance or refresh sequence without duplicating steps.

``` title="Child TS-LCU"
Lenovo Client Update
├─ Download Lenovo Modules
├─ Populate Repository
├─ 1st Pass - BIOS/Firmware
│   ├─ Install Applicable BIOS/Firmware
│   └─ Restart Computer                  [RebootMandatory = True]
├─ 2nd Pass - Drivers
│   └─ 2nd Pass - Drivers
└─ Final Pass Catch-All
    ├─ Set ApplicableUpdates Variable
    └─ 3rd Pass - Drivers                [ApplicableUpdates = True]
```

![Child TS-LCU task sequence structure in the Configuration Manager task sequence editor](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image4.jpg)

BIOS and firmware install first, followed by a reboot, and drivers install across the passes that follow. Firmware changes what the hardware presents to Windows, so drivers evaluated after that reboot are matched against the final state of the machine rather than the state it booted with. Running drivers first means re-evaluating them anyway once the firmware settles.

The driver work is split across passes for the same reason. Installing one driver can make another applicable, and a package that fails on the first attempt because a dependency was not yet present will often succeed on a second run against the same repository.

### Common Step Settings

Every **Run PowerShell Script** step in the child sequence uses the same configuration apart from the success codes:

| Setting | Value |
|---|---|
| Script source | Enter a PowerShell script |
| PowerShell execution policy | Bypass |
| Parameters | Empty |
| Output to task sequence variable | Unchecked |
| Success codes | `0`, or `0 3010` where noted |

Every step after the first depends on **Download Lenovo Modules** having run. None of them call `Import-Module`, and none need to: the module manifest lists its exports explicitly, and the install step writes to `%ProgramFiles%\WindowsPowerShell\Modules`, which is on the default `PSModulePath` in every new process. PowerShell resolves `Get-LnvUpdate` and friends to LCU on first use and imports the module automatically.

Each step also resolves the repository to `<_SMSTSMDataPath>\LenovoUpdates` on its own by reading the task sequence environment, which lands at `C:\_SMSTaskSequence\LenovoUpdates` and is removed when the sequence completes. Leave the **Parameters** field empty unless you want the repository somewhere else.

Leave **Continue on error** off on the module and repository steps. A missing module or an empty repository means every step after it has nothing to do, and failing there points at the real problem instead of burying it in a later step.

### Download Lenovo Modules

The gallery serves module content over an API that Windows PowerShell 5.1 can call directly, so this pulls the `.nupkg` down over `Invoke-RestMethod` and expands it into a versioned folder under the module root. That avoids `Install-Module`, which prompts to install the NuGet provider on a freshly imaged machine and blocks the step.

`Lenovo.Client.Scripting` is included alongside LCU since it is generally useful later in the sequence. Drop it from the `$modules` array if you only need updates.

``` powershell title="Download Lenovo Modules"
$modules = @(
    'Lenovo.Client.Update'
    'Lenovo.Client.Scripting'
)

# The gallery requires TLS 1.2, which Windows PowerShell 5.1 does not negotiate by default.
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

$moduleRoot = "$env:ProgramFiles\WindowsPowerShell\Modules"
Write-Output "Modules will be installed to $moduleRoot"

# Ensure the target root is on PSModulePath for this session.
if (-not ($env:PSModulePath -split ';' -contains $moduleRoot))
{
    $env:PSModulePath = $env:PSModulePath + ";$moduleRoot"
    Write-Output "Added $moduleRoot to PSModulePath"
}

foreach ($module in $modules)
{
    $params = @{
        Uri  = 'https://www.powershellgallery.com/api/v2/FindPackagesById()'
        Body = @{
            '$filter' = 'IsLatestVersion eq true'
            id        = "'$module'" # Extra quotes are required.
        }
    }
    $searchResult = Invoke-RestMethod @params
    if (-not $searchResult)
    {
        Write-Warning "$module was not found in the gallery. Skipping."
        continue
    }

    $installedVersion = Get-Module -Name $module -ListAvailable |
        Sort-Object -Property Version -Descending |
        Select-Object -First 1 -ExpandProperty Version

    if ($installedVersion -and [version]$installedVersion -ge [version]$searchResult.properties.version)
    {
        Write-Output "$module $installedVersion is already installed. Skipping."
        continue
    }

    # Download the module archive
    Invoke-RestMethod -Uri $searchResult.content.src -OutFile module.zip

    # Expand the archive to a versioned folder under the module root.
    $params = @{
        Path        = 'module.zip'
        Destination = $moduleRoot |
            Join-Path -ChildPath $module |
            Join-Path -ChildPath $searchResult.properties.version
        Force       = $true
    }
    Expand-Archive @params

    # Clean up redundant package files.
    Get-ChildItem -Path $params.Destination |
        Where-Object Name -In @(
            '_rels'
            'package'
            '[Content_Types].xml'
            "$module.nuspec"
        ) |
        Remove-Item -Recurse
    Write-Output "Installed $module $($searchResult.properties.version) to $($params.Destination)"

    # Remove downloaded archive
    Remove-Item -Path module.zip
}
```

Installing to `%ProgramFiles%\WindowsPowerShell\Modules` puts the module on the default `PSModulePath`, so every step that follows can call LCU cmdlets by name with no path handling.

!!! note
    This step needs the device to reach `www.powershellgallery.com`. If your imaging network blocks it, use the [original packaged method](#original-method-packaged-script), which delivers the module as CM content.

### Populate Repository

Builds the Update Retriever style repository for the machine type being imaged. Every later step reads from it, so this runs once and the passes that follow never hit Lenovo's servers again.

`-RebootTypes '0,3,5'` pulls everything; narrow it to `'0,3'` to skip packages that force a delayed reboot.

``` powershell title="Populate Repository"
#Requires -Version 5.1
#Requires -RunAsAdministrator

[CmdletBinding()]
param (
    [Parameter(HelpMessage = 'Target repository directory')]
    [string] $RepositoryPath,

    [Parameter(HelpMessage = 'Reboot types: 0=none, 3=requires reboot, 5=delayed forced reboot')]
    [string] $RebootTypes = '0,3,5'
)

$ErrorActionPreference = 'Continue'

try
{
    $tsEnv = New-Object -ComObject Microsoft.SMS.TSEnvironment
}
catch
{
    Write-Error "Unable to connect to the task sequence environment: $($_.Exception.Message)"
    exit 1
}

if (-not $RepositoryPath)
{
    $RepositoryPath = Join-Path -Path $tsEnv.Value('_SMSTSMDataPath') -ChildPath 'LenovoUpdates'
}
Write-Output "Repository path: $RepositoryPath"

try
{
    Get-LnvUpdatesRepo -RepositoryPath $RepositoryPath -RebootTypes $RebootTypes -ErrorAction Stop
}
catch
{
    Write-Error "Failed to build the repository: $($_.Exception.Message)"
    exit 1
}

# Get-LnvUpdatesRepo creates the repository folder before it downloads anything, so the
# folder existing proves nothing. database.xml is only written once the catalog is populated.
$database = Join-Path -Path $RepositoryPath -ChildPath 'database.xml'
if (-not (Test-Path -LiteralPath $database))
{
    Write-Error "Repository build did not produce '$database'. No packages were downloaded."
    exit 1
}

$packageCount = @(Get-ChildItem -LiteralPath $RepositoryPath -Directory -ErrorAction SilentlyContinue).Count
Write-Output "Repository built: $packageCount package folder(s), catalog present at $database."

exit 0
```

The check on `database.xml` is the part worth keeping if you rewrite this. `Get-LnvUpdatesRepo` creates the repository folder before it downloads anything, so a failed download leaves an empty directory behind. Every pass after this one would then read that directory, find nothing, and report "no updates found" as a success. Testing for the catalog file instead of the folder turns a silent no-op into a failed step.

### 1st Pass - BIOS/Firmware

#### Install Applicable BIOS/Firmware

Filters the repository to `BIOS` and `Firmware` packages and installs them. Two details matter here.

`-SaveBIOSUpdateInfoToRegistry` writes the pending BIOS update details to the registry, which ConfigMgr reads to surface the update in hardware inventory.

The script also sets a `RebootMandatory` task sequence variable and returns `3010` when at least one update needs a restart, giving the Restart Computer step that follows something to condition on.

``` powershell title="Install Applicable BIOS/Firmware"
#Requires -Version 5.1
#Requires -RunAsAdministrator

[CmdletBinding()]
param (
    [Parameter(HelpMessage = 'Repository path containing Lenovo update packages')]
    [string] $RepositoryPath
)

# Left at Continue on purpose: Install-LnvUpdate emits Write-Error and continues when it
# skips a package (e.g. signature check), and Stop would abort the whole run.
$ErrorActionPreference = 'Continue'
$exitCode = 0

try
{
    $tsEnv = New-Object -ComObject Microsoft.SMS.TSEnvironment
}
catch
{
    Write-Error "Unable to connect to the task sequence environment: $($_.Exception.Message)"
    exit 1
}

# Default the variable up front so a later step's condition always has something to evaluate.
$tsEnv.Value('RebootMandatory') = 'False'

if (-not $RepositoryPath)
{
    $RepositoryPath = Join-Path -Path $tsEnv.Value('_SMSTSMDataPath') -ChildPath 'LenovoUpdates'
}

# The repository is built by an earlier task sequence step. Test for the catalog rather
# than the folder: Get-LnvUpdatesRepo creates the folder before downloading, so a failed
# build leaves an empty directory that would otherwise pass as "no updates found".
$database = Join-Path -Path $RepositoryPath -ChildPath 'database.xml'
if (-not (Test-Path -LiteralPath $database))
{
    Write-Error "No repository catalog at '$database'. Confirm the repository step ran and succeeded."
    exit 1
}
Write-Output "Repository path: $RepositoryPath"

try
{
    $updates = @(Get-LnvUpdate -Repository $RepositoryPath -ScratchDirectory $RepositoryPath -ErrorAction Stop |
            Where-Object { $_.Type -in 'BIOS', 'Firmware' })
}
catch
{
    Write-Error "Failed to query the update repository: $($_.Exception.Message)"
    exit 1
}

if ($updates.Count -eq 0)
{
    Write-Output 'No applicable BIOS or firmware updates found.'
    exit 0
}

Write-Output "$($updates.Count) BIOS/firmware update(s) to install."

$results = @($updates | Install-LnvUpdate -Path $RepositoryPath -SaveBIOSUpdateInfoToRegistry -ExportToWMI)

foreach ($result in $results)
{
    $status = if ($result.Success) { 'SUCCESS' } else { "FAILED ($($result.FailureReason))" }
    Write-Output "$($result.ID) | $($result.Title) | $status | ExitCode=$($result.ExitCode) | PendingAction=$($result.PendingAction)"
}

$installed = @($results | Where-Object { $_.Success })
$failed = @($results | Where-Object { -not $_.Success })
# REBOOT_MANDATORY only. A SHUTDOWN pending action would power the machine off and break
# the task sequence, so it is deliberately not treated as a reboot here.
$rebootPending = @($installed | Where-Object { $_.PendingAction -eq 'REBOOT_MANDATORY' })

Write-Output "Summary: $($installed.Count) installed, $($failed.Count) failed, $($results.Count) attempted."

if ($rebootPending.Count -gt 0)
{
    $tsEnv.Value('RebootMandatory') = 'True'
    Write-Output "$($rebootPending.Count) update(s) require a reboot. RebootMandatory set to True."
    $exitCode = 3010
}

if ($failed.Count -gt 0)
{
    Write-Warning "$($failed.Count) update(s) failed. See %ProgramData%\Lenovo\Lenovo.Client.Update\History for details."
}

exit $exitCode
```

Set **Success codes** on this step to `0 3010`. Leaving it at `0` fails the step the moment a BIOS update asks for a restart.

`$ErrorActionPreference` stays at `Continue` on purpose. `Install-LnvUpdate` writes an error and moves on when it skips a package, such as a failed signature check. Setting `Stop` turns one skipped package into an aborted step.

!!! warning
    The reboot check matches `REBOOT_MANDATORY` only. Some Lenovo packages report a `SHUTDOWN` pending action, which powers the machine off rather than restarting it and strands the task sequence. Treating that as a reboot would break the deployment, so it is excluded.

#### Restart Computer

Add **Add** > **General** > **Restart Computer**, set it to boot into **The currently installed default operating system**, and condition it on the variable the previous step set:

- **Add Condition** > **Task Sequence Variable**
- **Variable:** `RebootMandatory`
- **Condition:** equals
- **Value:** `True`

![Restart Computer step conditioned on the RebootMandatory task sequence variable](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image5.jpg)

`RebootMandatory` is set to `False` at the top of the install script before anything else runs. A task sequence condition on a variable that was never defined does not evaluate the way you would expect, so giving it a default keeps this step predictable on machines that had no firmware to install.

### 2nd Pass - Drivers

Filters the same repository to driver packages and installs them. This runs after the firmware reboot, so the drivers evaluated here are matched against the hardware as the BIOS now presents it.

``` powershell title="2nd Pass - Drivers"
#Requires -Version 5.1
#Requires -RunAsAdministrator

[CmdletBinding()]
param (
    [Parameter(HelpMessage = 'Existing repository directory')]
    [string] $RepositoryPath
)

# Left at Continue on purpose: Install-LnvUpdate emits Write-Error and continues when it
# skips a package (e.g. signature check), and Stop would abort the whole run.
$ErrorActionPreference = 'Continue'

try
{
    $tsEnv = New-Object -ComObject Microsoft.SMS.TSEnvironment
}
catch
{
    Write-Error "Unable to connect to the task sequence environment: $($_.Exception.Message)"
    exit 1
}

if (-not $RepositoryPath)
{
    $RepositoryPath = Join-Path -Path $tsEnv.Value('_SMSTSMDataPath') -ChildPath 'LenovoUpdates'
}

# The repository is built by an earlier task sequence step. Test for the catalog rather
# than the folder: Get-LnvUpdatesRepo creates the folder before downloading, so a failed
# build leaves an empty directory that would otherwise pass as "no updates found".
$database = Join-Path -Path $RepositoryPath -ChildPath 'database.xml'
if (-not (Test-Path -LiteralPath $database))
{
    Write-Error "No repository catalog at '$database'. Confirm the repository step ran and succeeded."
    exit 1
}
Write-Output "Repository path: $RepositoryPath"

try
{
    $updates = @(Get-LnvUpdate -Repository $RepositoryPath -ScratchDirectory $RepositoryPath -ErrorAction Stop |
            Where-Object { $_.Type -eq 'Driver' })
}
catch
{
    Write-Error "Failed to query the update repository: $($_.Exception.Message)"
    exit 1
}

if ($updates.Count -eq 0)
{
    Write-Output 'No applicable driver updates found.'
    exit 0
}

Write-Output "$($updates.Count) driver update(s) to install."

$results = @($updates | Install-LnvUpdate -Path $RepositoryPath -ExportToWMI)

foreach ($result in $results)
{
    $status = if ($result.Success) { 'SUCCESS' } else { "FAILED ($($result.FailureReason))" }
    Write-Output "$($result.ID) | $($result.Title) | $status | ExitCode=$($result.ExitCode)"
}

$installed = @($results | Where-Object { $_.Success })
$failed = @($results | Where-Object { -not $_.Success })

Write-Output "Summary: $($installed.Count) installed, $($failed.Count) failed, $($results.Count) attempted."

if ($failed.Count -gt 0)
{
    Write-Warning "$($failed.Count) driver(s) failed. See %ProgramData%\Lenovo\Lenovo.Client.Update\History for details."
}

exit 0
```

This step runs unconditionally. It is the pass that does the bulk of the driver work, and the script already exits early with "No applicable driver updates found." when there is nothing to install, so a condition would only duplicate a check the script makes anyway.

The per-update lines written to standard output land in **smsts.log**, so the log shows which package produced which exit code without opening the LCU history file.

### Final Pass Catch-All

The last group splits the work in two: measure what the earlier passes left behind, then act on the answer. That measurement is the reason this is a group rather than one more install step. A blind second run would install nothing on most machines and still cost a full repository evaluation inside the install script; querying first turns the catch-all into a step that reports why it did or did not run, and leaves a line in **smsts.log** saying how many packages were still outstanding.

Nothing is downloaded again in either step. `Get-LnvUpdate` re-evaluates applicability against the machine as it now stands, which picks up a package that became applicable once an earlier driver landed as well as one that failed its first attempt.

#### Set ApplicableUpdates Variable

Re-queries the repository and sets `ApplicableUpdates` to `True` when anything is still pending. The filter is `IsApplicable` rather than a package type, so it will pick up any missing component.

``` powershell title="Set ApplicableUpdates Variable"
# Initialize TS environment
$tsEnv = New-Object -ComObject Microsoft.SMS.TSEnvironment

# Default the variable up front so a later step's condition always has something to evaluate.
$tsEnv.Value('ApplicableUpdates') = 'False'

# Set repository path
$RepositoryPath = Join-Path -Path $tsEnv.Value('_SMSTSMDataPath') -ChildPath 'LenovoUpdates'

# Query the repository in this session - Get-LnvUpdate runs under Windows PowerShell 5.1
$updates = @(Get-LnvUpdate -Repository $RepositoryPath -ScratchDirectory $RepositoryPath |
        Where-Object { $_.IsApplicable })

if ($updates.Count -gt 0)
{
    $tsEnv.Value('ApplicableUpdates') = 'True'
}

Write-Output "$($updates.Count) applicable update(s). ApplicableUpdates set to $($tsEnv.Value('ApplicableUpdates'))."
```

#### 3rd Pass - Drivers

Paste the same **2nd Pass - Drivers** script from above into this step unchanged. It reads the same repository and installs whatever is still applicable, so a package skipped on the second pass because a dependency was missing gets another attempt here.

This is the step that consumes the variable. Condition it so the catch-all is skipped entirely on a machine the earlier passes already finished:

- **Add Condition** > **Task Sequence Variable**
- **Variable:** `ApplicableUpdates`
- **Condition:** equals
- **Value:** `True`

Set **Success codes** to `0 3010`. The script returns `0` on its own, and the extra code leaves room for a package that reports a pending restart without failing the step.

![3rd Pass - Drivers step with success codes 0 3010 and the ApplicableUpdates condition](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image6.jpg)

!!! note
    On most machines the second pass covers everything, `ApplicableUpdates` comes back `False`, and this step is skipped. The catch-all costs a single repository query on the machines that do not need it.

## Original Method: Packaged Script

This is the method this post originally described, and it still works. The module and script ship as a CM package, so the task sequence never contacts the PowerShell Gallery, and the module version changes only when you update the package.

### Step 1: Prepare the Lenovo Client Update Module and Script

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

### Step 2: Create the Configuration Manager Package

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

### Step 3: Add the Run PowerShell Script Step to the OSD Task Sequence

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

Both methods record results in the same places.

Once the process completes, reference the **InstallHistory.json** located under **%ProgramData\Lenovo\Lenovo.Client.Update\History** for installation status on each update.

![](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image2.jpg)

The **smsts.log** will also track details for each update as they're processed, including a signature check.

![](https://cdrt.github.io/mk_blog/img/2026/lcu_configmgr_osd/image3.jpg)

## Final Notes

- **Reboots:** BIOS/firmware updates require reboots. The recommended method includes a conditional **Restart Computer** step; with the original method, add one after the update step.
- **Testing:** Run the TS on a physical Lenovo device to validate.
