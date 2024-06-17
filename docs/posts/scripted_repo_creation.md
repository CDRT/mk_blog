---
date: 
    created: 2023-03-13
authors:
    - Phil
    - Joe
categories: 
    - "2023"
title: Creating Local Repository Using PowerShell
---

![PowerShell](..\img/2023/scripted_repo_creation/ps_icon.png)

There are various scenarios where one might want to quickly generate a local repository of Lenovo updates that can be consumed by Thin Installer or System Update in a scripted manner. This article will describe a PowerShell script that can be leveraged to create a repository for a specified machine type and OS. A scenario where this script might be used will also be described.
<!-- more -->
The script, **Get-LnvUpdatesRepo.ps1**, can be found in the CDRT Library repository on GitHub [here](https://github.com/CDRT/Library). The parameters and switches used by the script are documented at the beginning of the script.

The child task sequence described in the first scenario can be downloaded [here](https://download.lenovo.com/cdrt/eval/GetLenovoUpdates.zip).

## Scenario 1

In the first scenario, Thin Installer will be leveraged in a Microsoft Configuration Manager Operating System Deployment task sequence to apply any applicable updates available for the machine type of the targeted system. For this approach, a PowerShell script will be executed in a child task sequence to orchestrate the creation of the repository with the desired updates which can be customized based on Package Types and Reboot Types.

### Child Task Sequence Workflow

The child task sequence will be added after the **Setup Windows and Configuration Manager** step in your parent Operating System Deployment task sequence

The top level group **Prepare Thin Installer** queries the device to determine if it is a ThinkPad, ThinkCentre or ThinkStation commercial PC product.
![Prepare Thin Installer](..\img/2023/scripted_repo_creation/image1.png)

A WQL query is used to check if the Operating System build is 22H1 or earlier. If it is, then Thin Installer is installed using a legacy Package in the 'Package' group:
![Package: Thin Installer](..\img/2023/scripted_repo_creation/image2.png)
![Package: Thin Installer](..\img/2023/scripted_repo_creation/image3.png)

Another WQL query is used to check if the Operating System build is 22H2 or later in the 'Winget' group. This group contains a step to install Thin Installer from the Winget repository. Windows 11 22H2 contains Winget automatically while earlier versions of Windows must install Winget from the Microsoft.DesktopAppInstaller package. If Winget is available in your environment on earlier OS builds, you can change the conditions on these groups and leverage the Winget install task instead.
![Winget: Thin Installer](..\img/2023/scripted_repo_creation/image4.png)

PowerShell script to install Thin Installer using Winget:

```powershell
$Winget = Get-ChildItem -Path (Join-Path -Path (Join-Path -Path $env:ProgramFiles -ChildPath "WindowsApps") -ChildPath "Microsoft.DesktopAppInstaller*_x64*\winget.exe")

try {
     & $Winget install --id Lenovo.ThinInstaller -h --accept-source-agreements --accept-package-agreements --log C:\ProgramData\Winget-InstallThinInstaller.log
}
catch {
    return $_.Exception.Message; Exit 1
}
```

PowerShell script **Get-LnvUpdatesRepo.ps1** will download current updates from Lenovo's servers and store on the device. Parameters are available to specify the repository path and filter updates to download by package types and reboot types. The command line used in this scenario is as follows:

```cmd
-RepositoryPath 'C:\ProgramData\Lenovo\ThinInstaller\Repository' -PackageTypes '1,2,3,4' -RebootTypes '0,3,5' -RT5toRT3
```

This filters the updates to include 1/Applications, 2/Drivers, 3/BIOS, and 4/Firmware packages; and updates with Reboot Type 0 (No reboot required), 3 (Reboot required but not forced), and 5 (Delayed forced reboot). The -RT5toRT3 parameter is a special case to be used in task sequences which will control the rebooting of the system after Thin Installer runs. It will cause the script to change Reboot Type 5 updates to Reboot Type 3 which will cause Thin Installer to not perform the reboot. **After these types of updates are applied, the system needs to be rebooted immediately to avoid possible damage to the system. This is taken care of by the task sequence.**

![Get Lenovo Updates](..\img/2023/scripted_repo_creation/image5.png)

!!! info ""
    The **All Updates** group contains the necessary parameters and Thin Installer command line to include Reboot Type 5 packages (BIOS/Firmware). The **Drivers** group will only download Reboot Type 0 and 3 packages (Drivers), and is disabled by default.

Once all content is downloaded to the device, 3 passes of Thin Installer (with a reboot in between) installs all updates to ensure the device is current.

### First Pass

![Run Thin Installer](..\img/2023/scripted_repo_creation/image6.png)

If a BIOS update is applicable, this package gets installed first using this Thin Installer command line

```cmd
ThinInstaller.exe /CM -repository "C:\ProgramData\Lenovo\ThinInstaller\Repository" -search A -action INSTALL -includerebootpackages 0,3 -packagetypes 3 -debug -noicon -noreboot -exporttowmi
```

### Second Pass

![Run Thin Installer](..\img/2023/scripted_repo_creation/image7.png)

Only drivers and apps are filtered for installation using this command line

```cmd
ThinInstaller.exe /CM -repository "C:\ProgramData\Lenovo\ThinInstaller\Repository" -search A -action INSTALL -includerebootpackages 0,3 -packagetypes 1,2 -debug -noicon -noreboot -exporttowmi
```

!!! info ""
   In some cases, typically with Thunderbolt, there may be a requirement that the latest driver needs to be installed *BEFORE* the firmware can be updated. This pass will ensure those drivers are installed before the firmware is installed in the next pass.

### Final Pass

![Run Thin Installer](..\img/2023/scripted_repo_creation/image8.png)

Firmware packages, such as Intel ME Firmware, are installed in the final pass using this command line

```cmd
ThinInstaller.exe /CM -repository "C:\ProgramData\Lenovo\ThinInstaller\Repository" -search A -action INSTALL -includerebootpackages 0,3 -packagetypes 2,4 -debug -noicon -noreboot -exporttowmi
```

!!! info ""
   In some cases there are drivers that do not become applicable until another driver has been installed. This final pass includes drivers to ensure those are covered for a complete installation.

### Summary

With this scenario, there is very little effort needed to manage multiple models. This approach does download the updates on each device being deployed which may be a redundant use of network bandwidth to the Internet. However, this approach works very well in a scenario where the device is being reimaged off-site.

This approach could be modified easily to incorporate a centralized network storage for the repository so that multiple devices could share the same content. The **Get-LnvUpdatesRepo.ps1** script could be called from an admin's machine to create repository folders for each machine type. The script in the task sequence could then be modified to call Thin Installer with a -repository parameter pointing to the correct machine type folder.

Normally, if BIOS and firmware updates are included which will force a reboot (reboot type 1) or force a delayed reboot (reboot type 5) then the task sequence will be interrupted. Most current Lenovo ThinkPad, ThinkCentre and ThinkStation products have switched to using Reboot Type 5 for BIOS updates. Therefore, the **Get-LnvUpdatesRepo.ps1** script has an option, **-RT5toRT3**, which changes the Reboot Type to 3 instead of 5. This, in combination with the **-noreboot** parameter of Thin Installer, allows the updates to be applied with Configuration Manager controlling the restart.

In a future article we will look at other scenarios that are made possible with the **Get-LnvUpdatesRepo.ps1** script and may even provide additional features and enhancements to the script.
