---
date: 2023-07-26
authors:
    - Phil
categories:
    - "2023"
title: Deploying Intel ME Firmware <br> Updates with Intune
---

Intel ME firmware have always been a struggle to keep up-to-date. This guide will walk through deploying these updates as a Win32 app.
<!-- more -->
## Preparing the Win32 App

Download the Intel ME Firmware package for your model from the support [site](https://pcsupport.lenovo.com). Using the Win32 Content Prep [Tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool), we'll convert the package to an .intunewin file to upload to Intune.

Example command:

```dos
IntuneWinAppUtil.exe -c "C:\IntuneWin32\Source\ME\" -s n3crg03w.exe -o C:\IntuneWin32\
```

## Adding the Win32 App

Once complete, login to the [Intune admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps) and add a new Win32 app.

Fill out the app information:

![AppInfo](\img/2023/update_intel_mefw/image1.jpg)

Enter the install command line:

```dos
n3crg03w.exe /VERYSILENT /DIR="C:\IntelMEFW" /PARAM="/silent"
```

uninstall command line:

!!! info ""
    Required but since you can't uninstall this type of update, we'll simply delete the directory the contents are extracted to

```dos
cmd.exe /c rmdir C:\IntelMEFW /s /q
```

Change the device restart behavior to **Intune will force a mandatory restart**. This will allow you to set restart grace periods and restart notifications when you assign it.

![ProgramInfo](\img/2023/update_intel_mefw/image2.jpg)

Set requirements. Refer to the package ReadMe for minimum supported operating systems.

![Requirements](\img/2023/update_intel_mefw/image3.jpg)

For detection rules, choose to use a custom script for detection. Below is an example PowerShell script that can be used:

```powershell
$DeployedFwVersion = "16.1.25.1932"
$LocalFwVersion = (Get-CimInstance -Namespace root/Intel_ME -ClassName ME_System).FWVersion

if([version]$LocalFwVersion -lt [version]$DeployedFwVersion) {
    Write-Output -InputObject "Firmware is outdated..."; Exit 3010
} else {
    Write-Output -InputObject "Firmware is up-to-date..."; Exit 0
}
```

Finish creating the Win32 app. Before assigning it, I'm going to create a Device Filter that only contains T14s Gen 3's so the app will only target these models.

The syntax for the filter is:

```dos
(device.model -startsWith "21BR") or (device.model -startsWith "21BS")
```

## Deploy/Monitor

For my testing, I assigned the app to All Devices and only including T14s Gen 3's using the device filter I just created.

![Assignment](\img/2023/update_intel_mefw/image4.jpg)

As I mentioned earlier, you can configure restart grace periods, restart notifications, and/or allowing the user to snooze the restart notification.

![Assignment](\img/2023/update_intel_mefw/image5.jpg)

Once the app finishes installing, a toast notification is presented prompting for a reboot.(This screenshot was taken from an L14)

![ToastNotification](\img/2023/update_intel_mefw/image6.jpg)
