---
date:
    created: 2024-11-07
    updated: 2025-02-28
authors:
    - Joe
categories:
    - "2024"
title: "Introducing: Lenovo Client Scripting Module 2.0.0"
---

![Lenovo Client Scripting Toolbox](https://cdrt.github.io/mk_blog/img\2024\intro_lcsm\toolbox.png)

**Update June 12, 2025: Version 2.3.0 is now available in the PowerShell Gallery!**

Back in January of 2024 we introduced the Lenovo Device Management Module. Today we are replacing it with the new Lenovo.Client.Scripting Module which is a PowerShell module that provides several useful cmdlets for making it easier to manage Lenovo commercial PCs. The module supports Lenovo's commercial portfolio of ThinkPad, ThinkCentre and ThinkStation products.
<!-- more -->

## Why the name change?

Some of you may be aware of another Lenovo product called Lenovo Device Manager.  This is a cloud-based management solution that we are continually enhancing to provide unique management capabilities for our commercial PCs as well as other devices like tablets and phones. This product is often referred to as LDM which started to be confused with LDMM, which is completely unrelated.

We also have plans to update some of our other PowerShell-based tools to fit under the same "Lenovo.Client" naming convention. More to come on that in the near future.

You can learn all about PowerShell modules here: [About Modules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules?view=powershell-7.3)

You can also find a complete reference for each of the available cmdlets in the module here:  [Lenovo.Client.Scripting Module](https://docs.lenovocdrt.com/guides/lcsm/lcsm_top)

## Installing Lenovo.Client.Scripting Module

The module itself is currently available for download here: [Lenovo.Client.Scripting.zip](https://download.lenovo.com/cdrt/tools/Lenovo.Client.Scripting_2.3.0.zip)

The module is also published in the PowerShell Gallery so it can easily be installed with the Install-Module command. To install from the PowerShell Gallery, simply run

``` PowerShell
Install-Module -Name Lenovo.Client.Scripting -Force
```

The zip file contains the module folder, Lenovo.Client.Scripting, which contains the Lenovo.Client.Scripting.psm1 and Lenovo.Client.Scripting.psd1 files plus the Public and Private sub-folders containing individual PowerShell scripts for the various functions. To manually install the module, there are two locations that the module folder can be copied to:

1. Per User: %UserProfile%\Documents\WindowsPowerShell\Modules
1. All Users: %ProgramFiles%\WindowsPowerShell\Modules

If you have the LnvDeviceManagement module installed already, we recommend you remove that module first.  The Lenovo.Client.Scripting module uses the same GUID with a higher version so it should supersede the earlier LDMM; however, it would be cleaner to remove LDMM first.

``` PowerShell
PS C:\> Remove-Module LnvDeviceManagement
```

To install the Lenovo.Client.Scripting module so that it is loaded into your PowerShell session, you need to call Import-Module first:

``` PowerShell
PS C:\> Import-Module Lenovo.Client.Scripting -Force
```

Then you can verify it is installed using the Get-Module command:

``` PowerShell
PS C:\> Get-Module Lenovo.Client.Scripting

ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Script     2.0.0      Lenovo.Client.Scripting             {Add-LnvSUCommandLine, Add-LnvSULogging, Export-LnvUpdateR...

```

## Using the Lenovo.Client.Scripting Module

All of the cmdlets from the Lenovo Device Management module are still present. A couple of new cmdlets have been added for which you can find details at [Lenovo.Client.Scripting Reference Guide](https://docs.lenovocdrt.com/guides/lcsm/lcsm_top). Let's see what a few of the cmdlets can do for us.

!!! info ""
    Many of the cmdlets require Internet access to provide data. Some may require running as Administrator.

### Get-LnvCVE

This cmdlet returns a list of the CVE identifiers that are listed as addressed vulnerabilities in the current BIOS update for the specified system. Only BIOS updates are covered by this cmdlet. A machine type can be passed as a parameter.  If no parameter is specified, the machine type of the running system will be used. CVE Data may not be available for all machine types.

``` PowerShell
PS C:\> Get-LnvCVE -MachineType 21DD
CVE-2023-31100
CVE-2022-4304
CVE-2022-40982
CVE-2023-28005
CVE-2022-36763
CVE-2022-36764
CVE-2022-36765
CVE-2022-27635
CVE-2022-40964
CVE-2022-36351
CVE-2022-38076
CVE-2021-42299
CVE-2021-38578
CVE-2022-48189
CVE-2022-34301
CVE-2022-34302
CVE-2022-34303
PS C:\>
```

### Find-LnvMachineType

But what if you don't know the machine type of a particular model? You can use this cmdlet to the possible machine types based on the friendly model name. Specify less detail in the model name if you are not finding the results you are looking for. This will increase the chance of a string match and will return more results. In some cases you may see multiple machine types for the same model. Some of these may be Intel vs. AMD models. If you are creating a script to run on a device and just need the machine type of that device, you can use the Get-LnvMachineType  (see [below](#lenovo-device-data)).

``` PowerShell
PS C:\> Find-LnvMachineType -ModelName 'ThinkPad L15 Gen 4'
ThinkPad L15 Gen 4 Type 21H3 21H4 = 21H3
ThinkPad L15 Gen 4 Type 21H3 21H4 = 21H4
ThinkPad L15 Gen 4 Type 21H7 21H8 = 21H7
ThinkPad L15 Gen 4 Type 21H7 21H8 = 21H8
PS C:\>
```

### Get-LnvAvailableBiosVersion

If you specify a machine type, the cmdlet will return the version of the  currently available BIOS update. If no machine type is specified, the cmdlet will use the running system's machine type and will compare the version of the currently available update to the version of the system and return an alert if the update is newer. The ```-Download``` switch can be used to trigger the download of the current update in either case and it will be stored in the current working directory.

``` PowerShell
PS C:\> Get-LnvAvailableBiosVersion -MachineType 21EY
Current available version: 1.08
PS C:\> Get-LnvAvailableBiosVersion -MachineType 21EY -Download                                                                               Current available version: 1.08                                                                                                               PS C:\> ls


    Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         11/2/2023   3:11 PM                0.0.5
d-----         9/30/2023  11:53 AM                EFI
d-----          5/7/2022   1:13 AM                PerfLogs
d-r---        10/24/2023   1:31 AM                Program Files
d-r---          5/7/2022   1:19 AM                Program Files (x86)
d-----          5/7/2022   1:17 AM                sources
d-r---        10/24/2023   1:30 AM                Users
d-----         11/2/2023   3:14 PM                Windows
-a----         11/2/2023   4:24 PM       17188096 n3ouj04w.exe
-a----         11/2/2023   4:13 PM          38031 taxonomy.txt


PS C:\>
```

### Get-LnvDriverPack

This cmdlet will download the SCCM Driver Pack based on the specified machine type, OS and OS build version. Tab completion can be used to select the OS build version in the correct format. The cmdlet will leverage the default browser for downloading the pack so the user can select the location to save the file to.

``` PowerShell
PS C:\> Get-LnvDriverPack -MachineType 21EY -WindowsVersion 11 -OSBuildVersion 22H2
```

### Lenovo Device Data

Sometimes you may be writing a script that needs to make use of a certain data element and you cannot easily remember the WMI query needed to get the value you are looking for.  There are several cmdlets that makes this task much easier.

``` PowerShell
PS C:\> Get-LnvMachineType
21DD

PS C:\> Get-LnvModelName
ThinkPad P1 Gen 5

PS C:\> Get-LnvProductNumber
21DDZA2EUS

PS C:\> Get-LnvSerial
8675309
PS C:\>
```

### Get-LnvUpdatesRepo

For instances where Update Retriever cannot be used to create the local repository or where full automation of the repository creation is desired, this cmdlet can be used instead. It can be customized and executed on a regular basis to get the latest update packages. This cmdlet is based on the PowerShell script that was documented in this blog article: [Create Local Repository Using PowerShell](https://blog.lenovocdrt.com/creating-local-repository-using-powershell)

!!! info ""
    One of the parameters of this cmdlet, -RT5toRT3, will generate a repository where the Reboot Type 5 updates are changed to Reboot Type 3 which modifies the XML Package Descriptor. A repository created this way with modified XML package descriptors will require you to use Thin Installer to process the updates in the repository. Commercial Vantage and Lenovo System Update will not recognize the modified updates.

``` PowerShell
PS C:\> Get-LnvUpdatesRepo -MachineTypes 21EY -WindowsVersion 11 -PackageTypes 3 -RebootTypes 5 -RepositoryPath c:\21EY

PS C:\>
```

## See Them All

There are currently 27 cmdlets included in the module and more will be added over time. To see the complete list and all the details for running each one of them, visit the reference guide [here](https://docs.lenovocdrt.com/guides/lcsm/lcsm_top).
