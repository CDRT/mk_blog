---
date:
    created: 2025-02-19
    updated: 2025-06-17
authors:
    - Phil
categories:
    - "2025"
title: SU Helper Use Cases in a Modern World
cover_image:
---

The SU Helper utility was introduced to programmatically trigger the System Update AddIn of Commercial Vantage. Let's see how we can leverage SU Helper in different scenarios across Microsoft's Endpoint Management solutions.

<!-- more -->

# Overview

This will be an ongoing article as new examples will continue to be introduced.

## Configuration Manager

### Run Scripts functionality

This example will demonstrate how to utilize Configuration Manager's ability to run PowerShell scripts.

!!! note
    Reference the MS docs [here](https://learn.microsoft.com/mem/configmgr/apps/deploy-use/create-deploy-scripts)

#### Scenario: On-Demand BIOS Update

In this scenario, we're going to demonstrate how to deploy a PowerShell script to a [co-managed](https://learn.microsoft.com/mem/configmgr/comanage/overview) Windows endpoint that will trigger the System Update AddIn of Commercial Vantage to filter and install only applicable BIOS updates, followed by presenting a toast notification to the end user prompting to install and reboot. Keep in mind, this example is more or less a "quick and dirty" solution but is intended to show expected behavior, where to trace the System Update log for installation status, and querying the Lenovo_Updates WMI class.

Follow the steps to [create a script](https://learn.microsoft.com/mem/configmgr/apps/deploy-use/create-deploy-scripts#create-a-script) and insert the below sample PowerShell code. Once created, approve it.

```powershell
# Define SU Helper variables
$basePath = "$env:ProgramFiles\Lenovo\SUHelper"
$suHelperPath = Join-Path -Path $basePath -ChildPath "SUHelper.exe"
$suParams = @('-autoupdate', '-packagetype', '3') # Specifying Package Type 3 to filter only BIOS updates (https://docs.lenovocdrt.com/guides/cv/suhelper/#-packagetype-string)

# Check if SUHelper.exe exists
if (-Not (Test-Path $suHelperPath))
{
    Write-Error "SUHelper.exe not found at $suHelperPath."
    exit 1
}

try
{
    # Fetch applicable updates
    $applicableUpdates = Get-CimInstance -Namespace root/Lenovo -ClassName Lenovo_Updates | Where-Object { $_.Status -eq "Applicable" }

    # Check for BIOS update
    $biosUpdateAvailable = $applicableUpdates | Where-Object { $_.Title -match "BIOS" }

    if (-Not $biosUpdateAvailable)
    {
        Write-Output "No BIOS update available."
        exit 0
    }
    else
    {
        Write-Output "BIOS update available to install. Triggering SU Helper."
    }

    # Start the SU Helper process
    $process = Start-Process -FilePath $suHelperPath -ArgumentList $suParams -NoNewWindow -PassThru
    $process.WaitForExit()

    if ($process.ExitCode -ne 0)
    {
        Write-Error "SUHelper.exe exited with code $($process.ExitCode)."
        exit 1
    }
}
catch
{
    Write-Error "Error occurred: $_"
    exit 1
}
```

![ConfigMgr-NewRunScript](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image1.jpg)

On a test device, assume Commercial Vantage and SU Helper is installed. For this demo, I'm going to manually launch Commercial Vantage and run a check for updates. Here you can see one recommended update, which is BIOS.

![CV-ApplicableBIOS](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image2.jpg)

I can also retrieve this by querying the **Lenovo_Updates** WMI class.

![WMI-ApplicableBIOS](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image3.jpg)

If you have [enabled logging](https://docs.lenovocdrt.com/guides/cv/commercial_vantage/), you can review the update candidate list in the System Update AddIn log which will also show you the package's reboot type. BIOS packages will be a type **5**.

![SULog](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image4.jpg)

Back in the ConfigMgr console, I'm going to select my test device and **Run Script** from the ribbon bar, choosing the **Trigger SU Helper** script approved earlier. Shortly after, I see the script has successfully run and that a BIOS update is available to install.

![ConfigMgr-ScriptRun](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image5.jpg)

Now for the user experience. BIOS updates are designed to present a prompt to the end user informing that this update will require a reboot. This is what the toast notification will look like, which can be customized with your company's branding as of version [10.2501.15.0](https://docs.lenovocdrt.com/guides/cv/#v102501150-january-2025).

![CV-BIOSInstallToast](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image7.jpg)

I'm going to click OK to proceed with the update. In the background, the BIOS update will be installed and staged for flashing upon the next reboot. Assuming all is well, I can open the latest System Update AddIn log and filter for **success** and see the BIOS package was installed successfully.

![SULog-BIOSSuccess](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image6.jpg)

Once the process completes, a second toast notification will be presented to the user informing that the system will reboot in 5 minutes.

![SULog-BIOSSuccess](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image8.jpg)

After the reboot completes, I can launch Commercial Vantage and see the update history now shows the BIOS update was installed successfully with a timestamp.

![CV-UpdateHistory](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image9.jpg)

If I query WMI, I can see the the property values have been updated as well

![WMI-BIOSSuccess](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image10.jpg)

## Intune

### Remediations

This scenario describes using Intune [Remediations](https://learn.microsoft.com/intune/intune-service/fundamentals/remediations) to trigger SU Helper.

#### Scenario: Silently Updating Drivers

In this scenario, we're going to deploy a PowerShell remediation script package to filter for drivers and install only reboot type 0 and 3 updates. These updates will install silently (requiring a reboot) and won't present any prompts to the user.

!!! info
    SU Helper [documentation](https://docs.lenovocdrt.com/guides/cv/suhelper/#-packagetype-string) for details regarding -packagetype and -reboottype parameters

The detection will check a few items, such as SU Helper presence and the last time the **updates_history.txt** file was last modified. This file gets updated every time Commercial Vantage scans for updates and can be found at this location:

```dos
C:\ProgramData\Lenovo\Vantage\AddinData\LenovoSystemUpdateAddin\session
```

1. In the Intune admin center, go to **Devices** > **Manage devices** > **Scripts and remediations**.

2. Click **Create** and enter a name and optionally a description.

3. Save and upload the Detection and Remediation script files from my [GitHub](https://github.com/philjorgensen/Intune/tree/main/Remediations/Apps/Commercial%20Vantage/SU%20Helper) and assign to a group, preferably one containing Lenovo devices.

At the time of this writing (June 11), I grabbed a system that hasn't been powered on in over 3 months so the threshold variable set in the detection script (30 days) should certainly trigger SU Helper.

![UpdatesRequired](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image11.png)

After triggering the remediation to run on the device and waiting several minutes, I see the timestamp for the updates_history text file has been modified and the SU Helper log output returned a 0 - Success [return code](https://docs.lenovocdrt.com/guides/cv/suhelper/#possible-return-codes). When I launched Commercial Vantage and checked the updates history, I see a handful of drivers that updated successfully.

![UpdatesRequired](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image12.png)

Back in the Intune admin center, adding the Pre/Post-remediation detection output columns, remediation did its job and the device is now up-to-date.

![UpdatesRequired](https://cdrt.github.io/mk_blog/img/2025/suhelper_use_cases/image13.png)