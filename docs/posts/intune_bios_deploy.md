---
date:
    created: 2019-03-11
    updated: 2024-01-14
authors:
    - Phil
categories:
    - "2019"
title: Deploying ThinkPad BIOS Updates With Intune
---

This walk-through will cover deploying ThinkPad BIOS updates with Intune. These are provided as standalone executables so adding them as a Win32 app will involve converting them to the .intunewin format using the [Win32 Content Prep Tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool).
<!-- more -->
## App Conversion

Create a working folder where the Win32 App Packaging Tool and BIOS packages will reside. Download the latest BIOS for your model system and save it to a working folder. In a PowerShell or Command Prompt, execute **IntuneWinAppUtil.exe** and follow the prompts to:

- **Specify the source folder** - This is the location where the BIOS package downloaded from the web is saved.
- **Setup File** - The BIOS package file name, i.e. **n3juj15w.exe**
- **Output folder** - Location where the converted app will drop.

Once this information is entered, you will see the tool validate the package parameters, encrypt the content, and generate the detection XML file. You'll now have a new file in the .intunewin format, which will need to be uploaded to Intune.

## Add the Win32 App

Login to the Microsoft Intune [admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps) to add a new Windows app. Choose **Windows app (Win32)** as the app type.

![App Type](..\img/2019/intune_bios_deploy//image1.jpg)

Select the newly created **.intunewin** file to upload

### App Information

Fill out the required app information and any optional fields.

![App Information](..\img/2019/intune_bios_deploy/image2.jpg)

### Program Information

On the **Program** page, this is where you'll specify the install/uninstall commands. ThinkPad BIOS updates are wrapped as Inno Setup packages that accept the /PARAM parameter and passes it to what is executed normally (Winuptp.exe -s). The uninstall command is required but in this case, not necessary since you can't uninstall a BIOS update.

Install command

```cmd
n3juj15w.exe /VERYSILENT /PARAM="-s"
```

Change the device restart behavior to **Intune will force a mandatory restart**. This will unlock additional options when you assign the app.

Leave the default return codes as-is. We'll need to add an additional code to verify a successful installation. Click **+Add**, set the value to **1** and select **Soft Reboot** for the code type.  For ThinkPad BIOS, a return code 1 indicates a successful BIOS update and no reboot (a silent install).  You can find a list of Winuptp return codes here.

![Program Information](..\img/2019/intune_bios_deploy/image3.jpg)

### Requirements

For **Requirements**:

- Operating system architecture - **64-bit**
- Minimum operating system - **Windows 10 1607**

Take it a bit further and configure an additional requirement rule to serve as a model check.

Requirement Type: **Registry**

Key path:

```cmd
HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS
```

Value name:

```cmd
SystemFamily
```

Value:

```cmd
ThinkPad P1 Gen 5
```

![Requirement Rule](..\img/2019/intune_bios_deploy/image4.jpg)

### Detection rules

Detection Rules can be handled several different ways. In this example, I'm choosing to look at the **BIOSVersion** value in the registry. The value in the screenshot below is after installing the latest BIOS update for my test system. This is what will be evaluated at the time of install, so if the client has an older BIOS installed, it should evaluate as **False** and proceed with the install.

![Detection Rule](..\img/2019/intune_bios_deploy/image5.jpg)

!!! info ""
    This detection method assumes a newer BIOS version is being deployed to a system on an older version. If you're attempting to deploy an older BIOS version, the rule will still evaluate as false and attempt to install the older version. If for some reason you're deploying an older BIOS version, make sure the **Secure Rollback Prevention** BIOS setting is disabled.

Key path

```cmd
HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS
```

Value name

```cmd
BIOSVersion
```

Value: Note the extra space between the 3 and ).

```cmd
N3JET37W (1.21 )
```

You can find the BIOS ID in the version release matrix on the support site.

![BIOS ID](..\img/2019/intune_bios_deploy/image6.jpg)

Values will vary across models so you'll need to confirm this data in the registry.

### Assign the App

Target a group for app assignment. If you're going to deploy multiple BIOS updates to different models, it may be a good idea to create a dynamic device group or device filter for each model and deploy its own BIOS to that group. An example query would be:

```cmd
(device.deviceModel -startsWith "20DC") -or (device.deviceModel -startsWith "20DD")
```

Since we configured Intune to force a mandatory restart, we can configure restart grace period options, such as notifying the user when the device will be restarted and a countdown timer leading up to the event.

![Assignment Settings](..\img/2019/intune_bios_deploy//image7.jpg)

### Client Side Experience

Once the app has been assigned, open the Company Portal on the client (if deployed as available) and choose to install the newly delivered app. If the app is required, the user should be presented with Windows toast notifications stating the update has been installed and requires a restart to complete installation.

![Restart Notification](..\img/2019/intune_bios_deploy//image8.jpg)

A restart countdown prompt will then be displayed before the machine is forced to restart.

![Restart Notification](..\img/2019/intune_bios_deploy//image9.jpg)

### Monitor

You can trace the workflow in the **IntuneManagementExtension.log** located under **C:\ProgramData\Microsoft\IntuneManagementExtension\Logs**. Highlighted below is during the app detection step and installation using the install commands specified.

![Detection](..\img/2019/intune_bios_deploy//image10.jpg)

![Instalation](..\img/2019/intune_bios_deploy//image11.jpg)

The device installation status in Intune may show failed (app not detected after install) due to the fact that the system has not restarted. This will return as "Installed" once the device has restarted and checks back into the Intune service.

!!! note
    If your laptops are encrypted with BitLocker, this needs to be taken into consideration. It's a best practice to suspend BitLocker prior to flashing the BIOS. ThinkPad BIOS has this check built into Winuptp so if the system is encrypted, Winuptp will suspend encryption behind the scenes before a reboot is triggered.
