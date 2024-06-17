---
date:
    created: 2017-09-08
authors:
    - Phil
categories:
    - "2017"
title: Dynamically Updating Think Product BIOS with ConfigMgr
---

![Holy Grail](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/holygrail.jpg)

!!! info ""
    What follows is a brief look at what is possible and not necessarily recommended for everyone.  Hopefully someone finds it useful.

Earlier at MMS this year (2017), a fantastic session on modern driver management in OS deployments was presented by Kim Oppalfens and Tom Degreef.  This method and what it entails can be found [**here**](http://www.oscc.be/sccm/osd/The-holy-grail-of-ConfigMgr-diver-management,-or-whatever-you-want-to-call-it/).
<!-- more -->
I was inspired by their session and wanted to see if this could work with Lenovo's BIOS updates in a similar manner.  The workflow is basically the same, with the key piece being the overridable task sequence variable in the Download Package Content step called **OSDDownloadDownloadPackages**.

Here's the layout of the Task Sequence:

![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image1.jpg)

## Creating the Package(s)
You'll need to download the latest BIOS for your model from Lenovo's support site and extract the contents to a source directory.  Here's the folder structure I use in my lab:

``` <Share>\OSD\BIOS\<first 4 characters of BIOS>\<version> ```

Example:

``` <Share>\OSD\BIOS\R06E\1.27 ```

The BIOS used here is for a ThinkPad Yoga 370.  You can find the SMBiosBiosVersion of your systems by running the following PowerShell command:

``` (get-wmiobject win32_bios).smbiosbiosversion ```

You can also find this on the BIOS Update Utility page for your system, near the bottom under Previous Version.

![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image2.jpg)

Back in the ConfigMgr console, create a standard package (no program).  Here's my Yoga 370 example:

[![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image3.jpg)](https://blog.lenovocdrt.com/img/2017/dynamic_bios_update/image3.jpg)

You can name these however you want as long as you have the 1st 4 characters of the BIOS in the Name field.  Also recommended to include the version of the BIOS and for easy access to future BIOS updates, I added the URL to the Update Utility page for the system in the Comment field.  Distribute the newly created Package to your distribution points.

## Dynamic BIOS PowerShell Script (Get-BIOSPackages.ps1)

Generate the XML by running the following commands (on the Site Server):

``` powershell
$SiteCode = $(Get-WmiObject -ComputerName "$ENV:COMPUTERNAME" -Namespace "root\SMS" -Class "SMS_ProviderLocation").SiteCode

Get-WmiObject -Class sms_package -Namespace root\sms\site_$SiteCode | Select-Object Name,PackageID,Version | Sort-Object -Property Name | Export-Clixml -Path "BIOSPackages.xml"
```

Create another Package named **Get-BIOSPackages**.  This will contain the source path of the PowerShell script that will dynamically match the system's BIOS to the appropriate PackageID as well as the BIOSPackages.xml that was generated above.  Credit goes to Kim Oppalfens for the original script, which I tweaked for the BIOS part.

!!! info ""
    If you are still using PowerShell v2, replace Get-CimInstance with Get-WmiObject

``` powershell
[xml]$Packages = Get-Content BIOSPackages.xml

# Environment variable call for task sequence only
   
$tsenv = New-Object -ComObject Microsoft.SMS.TSEnvironment
    
$BIOS = (Get-CimInstance -Namespace root\cimv2 -ClassName Win32_BIOS).SMBIOSBIOSVersion.Substring(0,4)

$ns = New-Object Xml.XmlNamespaceManager $Packages.NameTable

$ns.AddNamespace( "def", "http://schemas.microsoft.com/powershell/2004/04" )

$Xpathqry="/def:Objs/def:Obj//def:MS[contains(.,`"$BIOS`")]"

$Package = ($Packages.SelectNodes($xpathqry,$ns))

$PackageID = $Package.SelectNodes('def:S[contains(@N,"PackageID")]',$ns)

$tsenv.Value('OSDDownloadDownloadPackages') = $PackageID.InnerXML
```

## Task Sequence

1. **Run PowerShell Script** step, calling the **Get-BIOSPackages.ps1**

1. [**Download Package Content**](https://docs.microsoft.com/en-us/sccm/osd/understand/task-sequence-steps#BKMK_DownloadPackageContent) step:  Choose any Package here as it will be overridden anyway.
![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image4.jpg)

    Tick the box to save the path as a variable named **BIOS**.  What happens here is when there's a package match to download, it will be saved to the C:\_SMSTaskSequence\Packages directory while also setting the custom variable of **BIOS01**, which is the first package being downloaded.  Here's what it looks like in the smsts.log.

    ![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image5.jpg)

    !!! info ""
        Notice the PackageID of **PS100077**.  This matches the Yoga 370 Package that was created earlier.

1. **Run Command Line**: Flash ThinkPad BIOS

    !!! info ""
        If you have a combination of Think products, add a second and/or third step with the appropriate WMI query. See screenshots

    ![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image6.jpg)

    The command line here is simply executing winuptp64.exe with the silent switch, which is the BIOS flash utility.  The 64-bit version of winuptp should be found in just about all BIOS Update Utility packages for systems dating back to the Haswell line.

    In the **Start in:** field, enter **%BIOS01%** as this will tell winuptp64.exe to execute from the directory as explained above.

    ![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image7.jpg)

1. Task Sequence Variable to reboot system.  This will complete the flash operation.

    ![](https://cdrt.github.io/mk_blog/img/2017/dynamic_bios_update/image8.jpg)

    To finish the Task Sequence cleanly, I'm using the **SMSTSPostAction** variable with the value being **wpeutil reboot** (if in WinPE).  If in FullOS, use the native restart computer step.

**ThinkCentre Flash Command Line**

flash64.cmd /ign /sccm /quiet

SMSTSPostAction (If in WinPE) = wpeutil shutdown 

(If in FullOS) = cmd.exe /c shutdown -s -t 0 -f

!!! info ""
    This will not shut down the system to the point of having to physically push the power button to turn back on.  This just instructs the BIOS update to proceed to phase 2 of the flash process. The system will reboot and finish out the update.

**ThinkStation Flash Command Line**

flashx64.cmd

SMSTSPostAction (If in WinPE) = wpeutil shutdown 

(If in FullOS) = cmd.exe /c shutdown -s -t 0 -f

## Other things to consider

Depending on your scenario, if updating BIOS from full OS, don't forget to disable BitLocker (if applicable) prior to continuing and re-enable post flash.

Consider the following "gotcha" scenario:

Windows 7 is installed on a ThinkCentre, which is encrypted with BitLocker.  The protectors will not automatically enable after the flash completes and the system has rebooted as opposed to Windows 10, leaving the system unprotected.  This is because of the required shutdown in order to finish the flash.  A possible workaround is to add a Run Command Line step to add a RunOnce reg key to enable protectors before the shutdown.

``` cmd
reg add HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce /v EnableBitLocker /t REG_SZ /d "cmd.exe /c manage-bde -protectors -enable c:"
```
