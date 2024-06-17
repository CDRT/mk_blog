---
date: 
    created: 2023-08-10
authors: 
    - Phil
categories: 
    - "2023"
title: Enabling Logging for Commercial Vantage
---

As noted in our [docs](https://docs.lenovocdrt.com/#/cv/commercial_vantage), logging is not enabled by default any longer for Commercial Vantage. This is in regards to the System Update add-in. Historically, a log file is generated when a device checks for or installs updates.

Lenovo’s Product Security team wanted to ensure that enabling logs was a customer choice, and not something that was enabled by default without customer knowledge. This blog will provide example solutions for enabling logging for devices managed by Intune and ConfigMgr.
<!-- more -->
## Intune

There are several options for Intune managed devices. We'll leverage [**Remediations**](https://learn.microsoft.com/mem/intune/fundamentals/remediations) for this example. A simple detection script will check for the necessary registry name and its value. If the value is **False**, the remediation script will flip it to **True**.

Below is a sample detection that can be used:

```powershell
$Path = "HKLM:\SOFTWARE\WOW6432Node\Lenovo\SystemUpdateAddin\Logs"
$Name = "EnableLogs"
$Value = $true

try {
    $Registry = Get-ItemProperty -Path $Path -Name $Name -ErrorAction Stop | Select-Object -ExpandProperty $Name
    If ($Registry -eq $Value){
        Write-Output -InputObject "System Update logging enabled."
        Exit 0
    } 
    Write-Warning -Message "System Update logging not enabled."
    Exit 1
} 
catch {
    Write-Warning -Message "Could not enable System Update logging..."
    Exit 1
}
```

Followed by a sample remediation:

```powershell
Write-Output -InputObject "Enabling System Update Logging..."
$Path = "HKLM:\SOFTWARE\WOW6432Node\Lenovo\SystemUpdateAddin\Logs"
If (-not(Test-Path -Path $Path)) {
    New-Item -Path $Path -Force
}
Set-ItemProperty $Path EnableLogs -Value $true
```

Login to the Microsoft Intune [admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/DevicesMenu/~/remediations) and click **create script package**. Enter a name and select the detection and remediation script files to add. Choose to run the script in 64-bit PowerShell, configure the assignment and schedule. For our lab, I've set this to run hourly. Since this only applies to Lenovo devices, it may be worthwhile to create a device filter that only targets Lenovo branded systems. For example:

```dos
(device.manufacturer -eq "LENOVO")
```

![RemediationPackage](..\img/2023/cv_su_logging/image1.jpg)

Once the device retrieves the policy for Remediation scripts, track the **HealthScripts.log** for details. In this screenshot, I can see logging was enabled and verified in the Registry

![Remediation](..\img/2023/cv_su_logging/image2.jpg)

The next time the System Update feature of Commercial Vantage runs, you can find the log in the following location:

```dos
%ProgramData%\Lenovo\Vantage\AddinData\LenovoSystemUpdateAddin\logs
```

## ConfigMgr

For ConfigMgr managed devices, you can import a sample [Configuration Baseline](https://learn.microsoft.com/mem/configmgr/develop/compliance/about-configuration-baselines-and-configuration-items) that will accomplish the same result.

Download the **Baseline** [here](https://download.lenovo.com/cdrt/blog/CB_EnableSystemUpdateLogging.zip).

If you look at the properties of the Configuration Item, you'll see this is a simple Registry value setting type and a single compliance rule.

![CI](..\img/2023/cv_su_logging/image3.jpg)

![CI](..\img/2023/cv_su_logging/image4.jpg)

When you deploy the Baseline, be sure to check the options to **Remediate noncompliant rules when supported** and **Allow remediation outside the maintenance window**.

![CI](..\img/2023/cv_su_logging/image5.jpg)

Happy logging!
