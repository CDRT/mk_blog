---
date:
    created: 2024-10-16
authors:
    - Phil
categories:
    - "2024"
title: Deploying the Intel Processor Power Management Package
---

The Intel Processor Power Management (PPM) package is now available on the Lenovo support site. This will install a provisioning package on a supported system to tune power mode settings across AC/DC (Best Power Efficiency, Balanced, Best Performance). This article covers how to deploy the package using Microsoft Intune.

<!-- more -->

!!!note
    This package is not available in the System Update/Commercial Vantage catalog.

## Creating the Win32 App

Download the already-made .intunewin file and detection script from my GitHub

[https://github.com/philjorgensen/Intune/tree/main/Win32%20Apps/Intel_PPM](https://github.com/philjorgensen/Intune/tree/main/Win32%20Apps/Intel_PPM)

Login to the [Intune admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps) to create a new Win32 app.

Select the **Invoke-IntelPPMPackage.intunewin** app package file to upload. Fill out the required fields and any other data on the App information page.

On the Program page, we'll need to enter the install/uninstall commands.

**Install Command**

```dos
powershell.exe -ExecutionPolicy Bypass -File .\Invoke-IntelPPMPackage.ps1
```

**Uninstall Command**
```dos
powershell.exe -ExecutionPolicy Bypass -Command "Uninstall-ProvisioningPackage -AllInstalledPackages | Where-Object { $_.PackagePath.IndexOf('PPM') -ne -1 }"
```

Set the device restart behavior to **Determine behavior based on return codes**.

Change the 0 return code from Success to **Soft Reboot** and add 2 additional return codes:

- **1** = Failed
- **2** = Success

![ReturnCodes](https://cdrt.github.io/mk_blog/img/2024/intel_ppm_deploy/image1.jpg)

Select **64-bit** for the operating system architecture and **Windows 11 22H2** as the minimum operating system on the **Requirements** page.

Choose **Use a custom detection script** as the rule and specify the **Detect-IntelPPMPackage.ps1**.

There's no dependencies or supersedence that we need to configure so continue on to create the app. It

## Assigning the App

While the logic in the install script determines if the system is applicable for one of these packages, you should at least assign the app to a device group containing Lenovo systems.

## Experience

Since we configured additional return codes, the system should receive a toast notification informing of a restart to complete the installation.

![Toast](https://cdrt.github.io/mk_blog/img/2024/intel_ppm_deploy/image2.jpg)