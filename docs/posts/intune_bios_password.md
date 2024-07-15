---
date:
    created: 2024-07-15
authors:
    - Phil
categories:
    - "2024"
title: Changing the BIOS Supervisor Password with Intune <br> and the Think Bios Config Tool
---

For security purposes, there may be a requirement to change the supervisor password for BIOS access. There are a few ways to accomplish this, either programmatically using a couple lines of code or with the Think Bios Config Tool. The former would be easiest but does pose quite the risk by exposing your supervisor password in plain text, which is a big no-no. So instead, we're going to use the Think Bios Config Tool and an encrypted password file to tackle this scenario, which has become quite common these days it seems.

<!-- more -->

## Solution Overview

This solution uses a PowerShell script to change the BIOS supervisor password by invoking the Think Bios Config Tool, which passes a password file containing the encrypted supervisor password for authentication and password update.

## Preparing the Password File

First, download and extract the Think Bios Config Tool, which can be found [here](https://docs.lenovocdrt.com/guides/tbct/tbct_top). Launch the tool as Admin and tick the box beside **Supervisor password set on the target machine**. This will unlock additional options.

![TBCT](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image1.jpg)

Enter your current supervisor password in the **Enter password** field. In the **Enter encrypting key** field, either specify your own key (it can be anything you want) or click the **Generate a key** button to generate a random string. 

![TBCT](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image2.jpg)

Next, tick the box beside **Change Supervisor password**. This will unlock 2 extra fields where you specify the new supervisor password.

![TBCT](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image3.jpg)

Finally, click the **Create password change file** button. This will generate an .ini file in the directory where the tool was invoked.

!!! note
    Whenever you generate a password file, the name of the .ini will be the model of the machine it was generated from. You can rename it if you'd like.

![TBCT](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image4.jpg)

If you open the .ini file, you'll see a long string on a single line. This is the encrypting key that will be used for BIOS authentication so you're not passing the "real" one in plain text.

## Preparing the Win32 App

Copy the password file (.ini) and the Think Bios Config Tool (.hta) to a new directory.

Download the **Update-SVP.ps1** file from my [GitHub](https://github.com/philjorgensen/Intune) and save it to the directory containing the .hta and .ini.

!!! warning
    Before proceeding, you'll need to populate the $secretKey variable with the encrypting key you used to generate the password file

Using the Win32 Content Prep [Tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool), we're going to wrap these 3 items up and convert to an **.intunewin** file.

![TBCT](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image5.jpg)

An example command to create the **.intunewin** file

```dos
IntuneWinAppUtil.exe -c "C:\IntuneWin32\Source\tbct" -s Update-SVP.ps1 -o "C:\IntuneWin32\" -q
```

## Creating the Win32 App

### App Information

Add a new Windows app (Win32) in the [Intune admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps) and choose the **.intunewin** app package file you just created to upload. Fill out the required fields and click next.

![Win32](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image6.jpg)

### Program

Install command

```dos
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Update-SVP.ps1
```

Uninstall command

```dos
cmd.exe /c del %ProgramData%\Lenovo\ThinkBiosConfig\svp.status /s /q
```

Change **Device restart behavior** to **Determine behavior based on return codes**.

![Win32](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image7.jpg)

### Requirements

Operating system architecture: **64-bit**

Minimum operating system: **1607**

Add a new **Registry** requirement rule

Key path: **HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS**

Value name: **SystemManufacturer**

Registry key requirement: **String comparison**

Operator: **Equals**

Value: **LENOVO**

![Win32](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image8.jpg)

### Detection rules

Add a **File rule** type to check for a **.status** file

Path: **%ProgramData%\Lenovo\ThinkBiosConfig**

File or folder: **.svp.status**

Detection method: **File or folder exists**

![Win32](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image9.jpg)

No dependencies or supersedence is required so click through and assign the app to a group of devices.

## Experience

On a machine that's received the app assignment, a toast notification should be presented to complete a restart to finish the installation. This assumes a successful password change.

![Win32](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image10.jpg)

To verify this, look under **C:\ProgramData\Lenovo\ThinkBiosConfig** for a **svp.status** file. If you open the log file, you will see the tool found the password .ini file and successfully changed the password.

!!! note
    The log file name will change by machine. It's prefixed with the Machine Type Model followed by Serial.

![Win32](https://cdrt.github.io/mk_blog/img/2024/intune_bios_password/image11.jpg)

If the app installation failed, there can be a few scenarios that would cause this.

  - No supervisor password is set on the target machine
  - The encrypting key is incorrect

On a failed machine, the log will show **Access Denied** for password change if either of the above scenarios are true.
