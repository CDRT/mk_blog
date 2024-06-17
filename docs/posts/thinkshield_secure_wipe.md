---
date:
    created: 2021-12-08
    updated: 2023-08-03
authors:
    - Phil
categories:
    - "2021"
title: ThinkShield Secure Wipe
---

![Logo](\img/2021/thinkshield_secure_wipe/thinkshield.jpg)

## Overview

ThinkShield secure wipe is the successor to the ThinkPad Drive Erase Utility and is designed to provide the wipe out function of the SSD.

Although the Drive Erase Utility is still supported and provided as an external tool, ThinkShield secure wipe is fully integrated in the BIOS image and does not require any external tools.

Secure wipe can be executed locally by BIOS from the application menu of the Startup Boot Menu invoked by **F12** or remotely from OS through the WMI interface, which is what this post will be covering.
<!-- more -->
## Supported Systems

### All 2020 and later ThinkPad/ThinkCentre

### ThinkStation

| P Series |
|----------|
| P360 Tiny/Tower |
| P360 Ultra |
| P350 Tiny/Tower/Small Form Factor |
| P358 |
| P340 Tiny |

!!! warning
    These examples are intended to demonstrate a few different methods available to deploy the solution and not necessarily a "Best Practice". Adjust accordingly to fit your environment's needs. There is also no auditing/reporting provided by these methods.

## Requirements

The WMI service for ThinkShield secure wipe is available only when one of the following is set

- Supervisor Password (SVP)
- System Management Password (SMP)
- Hard Disk Password (HDP)

Sample PowerShell script that executes secure wipe on target system.

<https://github.com/CDRT/Library/tree/master/secure-wipe>

Save as **Invoke-ThinkShieldSecureWipe.ps1**

## Scenarios

The following examples will demonstrate how to invoke the ThinkShield secure wipe function with Microsoft Configuration Manager and Intune service

### Configuration Manager

#### SCENARIO 1a - Deploying using Run Scripts

Navigate to Software Library > Scripts > Create Script and either import **Invoke-ThinkShieldSecureWipe.ps1** or copy the contents into the script editor field

![CreateScript](\img/2021/thinkshield_secure_wipe/image1.jpg)

Specify the **EraseMethod**, **PasswordType**, and **Password** parameters. Details for each parameter is explained in the script header.

![ScriptParameters](\img/2021/thinkshield_secure_wipe/image2.jpg)

Complete the **Create Script** wizard and Approve it

![WizardCompletion](\img/2021/thinkshield_secure_wipe/image3.jpg)

Deploy to a single system or collection of systems. If successful, you should see a message stating the secure wipe succeeded and that the system needs to reboot to finish.

![Success](\img/2021/thinkshield_secure_wipe/image4.jpg)

#### SCENARIO 1b - Deploying as a Task Sequence

Create a new **Custom Task Sequence**. Edit the Task Sequence and add a **Run PowerShell Script** step. Tick the radio button **Enter a PowerShell script** and click **Edit Script...**

Browse to **Invoke-ThinkShieldSecureWipe.ps1** or copy the contents into the script editor.

In the **Parameters** field, enter the required parameters.

![Parameters](\img/2021/thinkshield_secure_wipe/image5.jpg)

Add a **Restart Computer** step to transition the system to secure wipe. In my lab, I deployed as an available Task Sequence and customized the notification texts.

![Restart](\img/2021/thinkshield_secure_wipe/image6.jpg)

### Intune

Package the **Invoke-ThinkShieldSecureWipe.ps1** as a Win32 app using the Microsoft Win32 Content Prep Tool.

![Create package](\img/2021/thinkshield_secure_wipe/image7.jpg)

Log into the Intune [admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps) and add a new **Win32 app**. Browse to the **Invoke-ThinkShieldSecureWipe.intunewin** file and add it for upload.

Specify App Information such as a **Name**, **Description**, and **Publisher**

![Application details](\img/2021/thinkshield_secure_wipe/image8.jpg)

Specify Program details

![Program details](\img/2021/thinkshield_secure_wipe/image9.jpg)

- Install Command

```powershell
powershell.exe -ExecutionPolicy Bypass -File ".\Trigger-ThinkShieldSecureWipe.ps1" -EraseMethod ATAN -PasswordType SVP -Password secretsvp
```

- Uninstall Command

```cmd
cmd.exe /c
```

**Device Restart Behavior**: Determine based on return codes

Set the OS architecture to **64-bit** and Minimum OS to **Windows 10 1607**

Add an additional requirement rule to check the system is Lenovo.

![Requirements](\img/2021/thinkshield_secure_wipe/image10.jpg)

- **Registry Type**
  - **Key Path**: HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS
  - **Value Name**: SystemManufacturer
  - **Key Requirement**: String Comparison
  - **Operator**: Equals
  - **Value**: LENOVO

Set the detection rule to check the presence of a **File**

![Detection](\img/2021/thinkshield_secure_wipe/image11.jpg)

This file will be created automatically when the script is run.

- **Path**: %ProgramData%\Lenovo\ThinkShield
- **File or folder**: SecureWipe.tag
- **Detection method**: File or folder exists

Deploy the app to a group. In my testing, I deployed as available and installed through the Company Portal.  After a successful install, a toast notification is presented instructing for the reboot.

![Toast notification](\img/2021/thinkshield_secure_wipe/image12.jpg)

Once the system has restarted, secure wipe will trigger.

![Secure wipe at POST](\img/2021/thinkshield_secure_wipe/image13.jpg)
