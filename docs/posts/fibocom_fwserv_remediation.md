---
date:
    created: 2025-01-17
authors:
    - Thad
categories:
    - "2025"
title: "Remediate Fibocom FwSwitchService on Devices Without WWAN Card"
---

After a successful Operating System Deployment (OSD), customers have noted the Fibocom FWSwitchService running on devices without a Fibocom WWAN Card.  In this blog post, we provide the cause and remediation for this issue.
<!-- more -->

## Cause

The FwUpdateDriver.inf in the SCCM Driver Pack is listed as boot critical, forcing it to be added to the driver store and installed to the operating system during OSD.

## Impacted Models

    ThinkPad T14s Gen 4
    ThinkPad T16 Gen 2
    ThinkPad X1 Carbon Gen 11
    (In the future, this list may expand as other models are identified.)

## Remediation

Apply a Configuration Manager Configuration Baseline with a Configuration Item to identify and resolve impacted devices.  We have exported the Configuration Baseline, which contains the Configuration Item, from an environment as CB – Fibocom FwSwitchService Fix.cab and are providing it in a ZIP file.  First, extract the .ZIP file and and then import the CB – Fibocom FwSwitchService Fix.cab into Configuration Manager.  Below is an explanation of the Configuration Baseline and Configuration Item.

[**Download the Configuration Baseline here**](https://download.lenovo.com/cdrt/blog/CB_FixFibocomFWSwitchService.zip)

The Configuration Item contains one script setting type consisting of detection/remediation scripts.  These scripts will detect if the device has Fibocom hardware, if the FwSwitchService is running, and will remediate if the hardware is not found and the service is running.

### Settings - Type Script

    Name - Detect Fibocom FWSwitchService
    Setting type - Script
    Data type - Boolean

### Supported Platforms

    All Windows 10 (ARM64)
    All Windows 10 (64-Bit)
    All Windows 11 (ARM64)
    All Windows 11 (64-bit)

### Discovery Script

Detect listed Fibocom devices.  If any of the listed Fibocom devices are found, return false to keep the FWSwitchService enabled on the device.  If none of the listed Fibocom devices are found and the FWSwitchService service is enabled, return true to be remediated.  If no listed Fibocom devices are found and the FWSwitchService service is disabled, return false as the service is not running.

```PowerShell
$FiboFWDev = Get-PnpDevice | Where-Object {$_.HardwareID -contains "MBFW\{ba3ec007-246e-4a4c-ad06-ee9342198298}" -or $_.HardwareID -contains "MBFW\{66e99d5f-c11d-4438-811f-6d816b7391bb}" -or $_.HardwareID -contains "MBFW\{65119331-831e-45f1-b036-fb8ac7c445dd}" -or $_.HardwareID -contains "MBFW\{7ff0c42a-1a08-493f-bb35-9a8c86dbb588}" -or $_.HardwareID -contains "MBFW\{536bfd93-86ab-4a6c-9ab0-0afc7ee4c11e}" -or $_.HardwareID -contains "MBFW\{81648c03-219b-434d-867d-7c262fea3f97}"}
If($Null -eq $FiboFWDev){
    $FiboFWServ = Get-Service -Name FirmwareSwitchService -ErrorAction SilentlyContinue
    If($FiboFWServ){
        If($FiboFWServ.StartType -eq 'Disabled'){
            Return $False
        }
        Else{
            Return $True
        }
    }
    Else{
        Return $False
    }
}
Else{
    Return $False
}
```

### Remediation Script

When a remediation request is attempted, the device should search through all the installed drivers and uninstall the Fibocom driver with the FWSwitchService.exe.  Upon uninstall, the service, if running, will stop and be set to disabled.  On the next reboot, the FWSwitchService service will be removed from the services snapin.

```PowerShell
$GCI = Get-ChildItem -Path "C:\Windows\Inf" -Name -Include *.inf
ForEach ($File in $GCI){$Content = Get-Content -Path "C:\Windows\Inf\$file"
    If($Content -match "FWSwitchService.exe"){
        Start-Process -FilePath "C:\windows\system32\pnputil.exe" -ArgumentList "/delete-driver $file /uninstall /force" -WindowStyle Hidden
    }
}
```

### Deploying the Baseline

When ready to deploy the baseline, be sure to check the box to remediate noncompliant rules when supported.  It is suggested to test this on a small collection of T14s Gen 4, X1 Carbon Gen 11, and T16 Gen 2 devices where the collection contains a mixture of devices both with and without the Fibocom WWAN card and review the impact.

![Baseline Deploy](https://cdrt.github.io/mk_blog/img/2025/fibocom_fwserv_remediation/image1.jpg)

### Monitoring

In the console, navigate to the Monitoring workspace and select the Deployments node.  Here you can review the status of the baseline.