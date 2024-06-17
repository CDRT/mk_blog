---
date:
    created: 2020-10-14
    updated: 2024-04-12
authors:
    - Phil
categories:
    - "2020"
title: Autopilot + System Update = Latest drivers
---

This post walks you through updating your Think product's drivers during [Windows Autopilot](https://docs.microsoft.com/en-us/mem/autopilot/windows-autopilot) using Lenovo System Update.

Ideally, you'll want the most current drivers installed on the device prior to the user's first sign-in.  
<!-- more -->
## Prerequisites

- ~~Latest version of [System Update](https://support.lenovo.com/downloads/ds012808-lenovo-system-update-for-windows-10-7-32-bit-64-bit-desktop-notebook-workstation)~~ Script will install System Update via Microsoft's [Winget PowerShell Module](https://www.powershellgallery.com/packages/Microsoft.WinGet.Client/)
- Microsoft's Win32 Content Prep [tool](https://github.com/Microsoft/Microsoft-Win32-Content-Prep-Tool)
- Sample PowerShell script that performs the following:
  - Installs PowerShell 7
  - Installs Microsoft.Winget.Client PowerShell module, which allows us to install System Update via Winget
  - Sets the AdminCommandLine registry value that will:
    - Download/install type 3 packages (drivers only). More info on package types can be found in the updated Deployment [Guide](https://docs.lenovocdrt.com/#/su/su_dg).
    - Writes the installation status of each update to WMI.
    - Configures the System Update GUI
    - Disables the default scheduled tasks created by System Update.
    - Sets a custom scheduled task for System Update to run.

Save the below as **Invoke-SystemUpdate.ps1**: <https://github.com/philjorgensen/Intune/blob/main/Autopilot/System%20Update/Invoke-SystemUpdate.ps1>

```powershell title="Invoke-SystemUpdate.ps1"
<# 
DISCLAIMER: 

These sample scripts are not supported under any Lenovo standard support   

program or service. The sample scripts are provided AS IS without warranty   

of any kind. Lenovo further disclaims all implied warranties including,   

without limitation, any implied warranties of merchantability or of fitness for   

a particular purpose. The entire risk arising out of the use or performance of   

the sample scripts and documentation remains with you. In no event shall   

Lenovo, its authors, or anyone else involved in the creation, production, or   

delivery of the scripts be liable for any damages whatsoever (including,   

without limitation, damages for loss of business profits, business interruption,   

loss of business information, or other pecuniary loss) arising out of the use   

of or inability to use the sample scripts or documentation, even if Lenovo   

has been advised of the possibility of such damages.  
#> 

<#
.SYNOPSIS
    Script to install and configure Lenovo System Update. 

.DESCRIPTION
    Script will install Lenovo System Update and set the necessary registry subkeys and values that downloads/installs 
    reboot type 3 packages on the system. Certain UI settings are configured for an optimal end user experience.
    The default scheduled task created by System Update will be disabled. A custom scheduled task for System Update will be created.
    
.NOTES
    FileName: Invoke-SystemUpdate.ps1
    Author: Philip Jorgensen

    Created:    2023-10-10
    Change:     2024-04-15
                    Switch to winget installation method using PowerShell 7 and the Microsoft.Winget.Client module
                    Add logging
                    Add soft reboot code to finish driver installation for drivers that may require it
#>

$LogPath = Join-Path -Path (Join-Path -Path $env:ProgramData -ChildPath "Lenovo") -ChildPath "SystemUpdate"
Start-Transcript -Path $LogPath\Autopilot-SystemUpdate.log

<# 
    Credit to Andrew Taylor
    https://github.com/andrew-s-taylor/public/blob/main/Powershell%20Scripts/Intune/deploy-winget-during-esp.ps1
#>
#GitHub API endpoint for PowerShell releases
$githubApiUrl = 'https://api.github.com/repos/PowerShell/PowerShell/releases/latest'

#Fetch the latest release details
$release = Invoke-RestMethod -Uri $githubApiUrl

#Find asset with .msi in the name
$asset = $release.assets | Where-Object { $_.name -like "*msi*" -and $_.name -like "*x64*" }

#Get the download URL and filename of the asset (assuming it's a MSI file)
$downloadUrl = $asset.browser_download_url
$filename = $asset.name

#Download the latest release
Invoke-WebRequest -Uri $downloadUrl -OutFile $filename

#Install PowerShell 7
Start-Process msiexec.exe -Wait -ArgumentList "/I $filename /qn"

#Start a new PowerShell 7 session
$pwshExecutable = "C:\Program Files\PowerShell\7\pwsh.exe"

#Run a script block in PowerShell 7
& $pwshExecutable -Command {
    $provider = Get-PackageProvider -Name NuGet -ErrorAction Ignore
    if (-not($provider))
    {
        Write-Host "Installing provider NuGet"
        Find-PackageProvider -Name NuGet -ForceBootstrap -IncludeDependencies
    }
}
& $pwshExecutable -Command {
    Install-Module -Name Microsoft.Winget.Client -Force -AllowClobber
}
& $pwshExecutable -Command {
    Import-Module -Name Microsoft.Winget.Client
}
& $pwshExecutable -Command {
    Repair-WinGetPackageManager
}
& $pwshExecutable -Command {
    Install-WinGetPackage -Id Lenovo.SystemUpdate
}

#Set SU AdminCommandLine
$Key = "HKLM:\SOFTWARE\Policies\Lenovo\System Update\UserSettings\General"
$Name = "AdminCommandLine"
$Value = "/CM -search A -action INSTALL -includerebootpackages 3 -noicon -noreboot -exporttowmi"

#Create subkeys if they don't exist
if (-not(Test-Path -Path $Key))
{
    New-Item -Path $Key -Force | Out-Null
    New-ItemProperty -Path $Key -Name $Name -Value $Value | Out-Null
}
else
{
    New-ItemProperty -Path $Key -Name $Name -Value $Value -Force | Out-Null
}
Write-Host "AdminCommandLine value set"

#Configure System Update interface
$Key2 = "HKLM:\SOFTWARE\WOW6432Node\Lenovo\System Update\Preferences\UserSettings\General"
$Values = @{

    "AskBeforeClosing"     = "NO"

    "DisplayLicenseNotice" = "NO"

    "MetricsEnabled"       = "NO"
                             
    "DebugEnable"          = "YES"
}

if (Test-Path -Path $Key2)
{
    foreach ($Value in $Values.GetEnumerator() )
    {
        New-ItemProperty -Path $Key2 -Name $Value.Key -Value $Value.Value -Force
    }
}
Write-Host "System Update GUI configured"

<# 
Run SU and wait until the Tvsukernel process finishes.
Once the Tvsukernel process ends, Autopilot flow will continue.
#>
$systemUpdate = "$(${env:ProgramFiles(x86)})\Lenovo\System Update\tvsu.exe"
& $systemUpdate /CM

Write-Host "Execute System Update and search for drivers"

#Wait for tvsukernel to initialize
Start-Sleep -Seconds 15
Wait-Process -Name Tvsukernel
Write-Host "Drivers installed"

#Disable the default System Update scheduled tasks
Get-ScheduledTask -TaskPath "\TVT\" | Disable-ScheduledTask
Write-Host "Default scheduled tasks disabled"

<# 
Disable Scheduler Ability.  
This will prevent System Update from creating the default scheduled tasks when updating to future releases.
#> 
$schdAbility = "HKLM:\SOFTWARE\WOW6432Node\Lenovo\System Update\Preferences\UserSettings\Scheduler"
Set-ItemProperty -Path $schdAbility -Name "SchedulerAbility" -Value "NO"

#Create a custom scheduled task for System Update
$taskActionParams = @{
    Execute  = $systemUpdate
    Argument = '/CM'
}
$taskAction = New-ScheduledTaskAction @taskActionParams

#Adjust to your requirements
$taskTriggerParams = @{
    Weekly     = $true
    DaysOfWeek = 'Monday'
    At         = "9am"
}
$taskTrigger = New-ScheduledTaskTrigger @taskTriggerParams
$taskUserPrincipal = New-ScheduledTaskPrincipal -UserId 'SYSTEM'
$taskSettings = New-ScheduledTaskSettingsSet -Compatibility Win8

$newTaskParams = @{
    TaskPath    = "\TVT\"
    TaskName    = "Custom-RunTVSU"
    Description = "System Update searches and installs new drivers only"
    Action      = $taskAction
    Principal   = $taskUserPrincipal
    Trigger     = $taskTrigger
    Settings    = $taskSettings
}
Register-ScheduledTask @newTaskParams -Force | Out-Null
Write-Host "Custom scheduled task created"
Write-Host "Exiting with a 3010 return code for a soft reboot to complete driver installation."
Stop-Transcript
Exit 3010
```

## Preparing the Win32 App (Old method)

Once all pre-requisites are downloaded to a source location, run the Content Prep tool to package the content as an **.intunewin** package. A sample command would be:

```cmd
IntuneWinAppUtil.exe -c "C:\SU\" -s "Configure-TVSUandScheduledTask.ps1" -o "C:\SU\output"
```

![Create package](https://cdrt.github.io/mk_blog/img/2020/ap_su/image1.jpg)

## Preparing the Win32 App (New method)

Download the **Invoke-SystemUpdate.intunewin** app from GitHub *or*

Using the Win32 Content Prep tool, create your own .intunewin file. Example:

```cmd
intunewinapputil.exe -c .\SU -s Invoke-SystemUpdate.ps1 -o .\ -q
```

## Add Win32 App

Add a new Win32 app in [Intune](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps). Select the .intunewin app created earlier and click ok to upload.

Fill out the necessary information for each section.

![App information](https://cdrt.github.io/mk_blog/img/2020/ap_su/image2.jpg)

For the Install command enter the following:

```cmd
powershell.exe -ExecutionPolicy Bypass -File .\Invoke-SystemUpdate.ps1
```

Uninstall command to uninstall System Update:

```cmd
"%ProgramFiles(x86)%\Lenovo\System Update\unins000.exe" /SILENT
```

![Install command](https://cdrt.github.io/mk_blog/img/2020/ap_su/image3.jpg)

Additional Requirement type: **Registry**

- Key path: **HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS**
- Value name: **SystemManufacturer**
- Registry key requirement: **String comparison**
- Operator: **Equals**
- Value: **LENOVO**
This ensures the app will only run on Lenovo systems

![Requirement](https://cdrt.github.io/mk_blog/img/2020/ap_su/image4.jpg)

Detection rule type: **File**

![Detection rules](https://cdrt.github.io/mk_blog/img/2020/ap_su/image5.jpg)

- Path: **%ProgramData%\Lenovo\SystemUpdate\sessionSE**
- File or folder: **update_history.txt**
- Detection method: **File or folder exists**

![Detection rule](https://cdrt.github.io/mk_blog/img/2020/ap_su/image6.jpg)

The update_history.txt is generated since we're specifying the **-exporttowmi** switch in the AdminCommandLine.  Since the system will be going through Autopilot for the first time, this obviously won't be present.

Assign the app to a group containing Autopilot registered devices.

If you already have an Enrollment Status Page profile configured, add this app to the list of selected apps that are required to install before the device can be used.  This ensures System Update completes before proceeding to the next phase.

![Required app](https://cdrt.github.io/mk_blog/img/2020/ap_su/image7.jpg)

## Viewing the Results

A look through the IntuneManagementExtension.log, you'll see the update_history.txt file was not detected

![Log](https://cdrt.github.io/mk_blog/img/2020/ap_su/image8.jpg)

Several minutes later, it's now detected

![Log](https://cdrt.github.io/mk_blog/img/2020/ap_su/image9.jpg)

You can then run the following PowerShell command to see which updates were installed

```powershell
Get-ChildItem -Path C:\ProgramData\lenovo\SystemUpdate\sessionSE\update_history.txt | Select-String -SimpleMatch "Success" | fl Line
```

The screenshot below shows the results from a ThinkPad T480s preloaded with Windows 10 1903. 13 drivers updated successfully!

![Results](https://cdrt.github.io/mk_blog/img/2020/ap_su/image10.jpg)
