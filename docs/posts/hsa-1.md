---
date:
    created: 2020-06-19
    updated: 2024-04-10
authors:
    - Thad
categories:
    - "2020"
title: Hardware Support Apps without Microsoft Store
---

With versions of Windows 10 since 1809, Microsoft has introduced the concept of Modern Drivers.  These new drivers have a few requirements:

* Declarative:  The driver must be INF installable with no co-installers
* Componentized:  The driver must support the architecture of having a base driver with optional extension drivers for customizations above the base functionality
* Hardware Support Apps:  Any software required for working with the device must be in the form of a UWP app available from the Microsoft Store and will be associated to this driver in the INF.
<!-- more -->
One intent of these items is to simplify in-place upgrades.  With this approach, if a Windows 10 device is upgraded to a new build by Windows Update and any new drivers are required, Windows Update can install the driver and it can trigger the software to be installed automatically from the Microsoft Store.

But what happens if you have blocked access to the Microsoft Store?  You will end up with a driver installed for a component without the software to control the device.  Keep in mind that not all device drivers will require a software component.  However, for those that do you may need them to get the full user experience from your device.

So how do you deploy these Hardware Support Apps (HSAs) without using the Microsoft Store?  This is where Lenovo's Hardware Support App Packs come in.

Starting in late June of 2020 you will begin to see these show up on the Lenovo Support site on the Drivers & Software page for the specific model, under the Enterprise Management component.  These will be available for the new products just launched and going forward.  The HSA Packs are similar to our SCCM Driver Packs.  They contain just the source files to install the apps for specific models.  One difference between the two is that the HSA Packs will typically need to be updated much less frequently.  With this architecture, the driver may change several times and continue to use the same HSA.

The HSA Pack will be a self-extracting executable file like our SCCM Driver Packs.  When you extract one you will get a folder structure with each app's source files contained in their own folder.  By default a folder with a random name will be used to prevent any kind of symlink vulnerability.

![HSA directory](https://cdrt.github.io/mk_blog/img/2020/hsa_directory.png)

### Scripted Install of Hardware Support Apps during a Task Sequence

We have created a script to read JSON manifest files provided in the HSA packs.  The script can be leveraged in MEM/SCCM or MDT.  Below is the PowerShell script you can use for installation.

```Powershell title="Install-HSA.ps1"

################################################################################
##                                                                            ##
##      Title: Install-HSA.ps1                                                ##
##  Publisher: Lenovo                                                         ##
##    Version: 1.03                                                           ##
##       Date: 2024-04-10                                                     ##
##                                                                            ##
##                              Legal Disclaimer                              ##
##                                                                            ##
##  The sample scripts are not supported under any Lenovo standard support    ##
##  program or service. The sample scripts are provided AS IS without         ##
##  warranty of any kind. Lenovo further disclaims all implied warranties     ##
##  including, without limitation, any implied warranties of                  ##
##  merchantability or of fitness for a particular purpose. The entire risk   ##
##  arising out of the use or performance of the sample scripts and           ##
##  documentation remains with you. In no event shall Lenovo, its authors,    ##
##  or anyone else involved in the creation, production, or delivery of the   ##
##  scripts be liable for any damages whatsoever (including, without          ##
##  limitation, damages for loss of business profits, business                ##
##  interruption, loss of business information, or other pecuniary loss)      ##
##  arising out of the use of or inability to use the sample scripts or       ##
##  documentation, even if Lenovo has been advised of the possibility of      ##
##  such damages.                                                             ##
##                                                                            ##
################################################################################


<#
.SYNOPSIS

    List the names of or install Hardware Support Applications (HSA) by 
    reading manifest files located in the subdirectories.

.DESCRIPTION

    By reading manifest files found in the subdirectories of a Hardware Support 
    Applications Pack, this script allows for the deployment of one, many, or 
    all Hardware Support Applications for a given model.  All installations are 
    against an offline installation of Windows 10.

.PARAMETER LIST

    Used to list the names of all Hardware Support Applications in the 
    subdirectories.  Returns the list of the Hardware Support 
 Applications names to the screen.

    -LIST

.PARAMETER EXPORT

    Used to list the names of all Hardware Support Applications in the 
    subdirectories.  Returns the list of names to the Export_{Date}{Time}.txt 
    file in the directory from which the script was executed.

    -LIST -EXPORT

.PARAMETER OFFLINE

    Used to install the Hardware Support Applications.

    -OFFLINE {-ALL, FILE '{FileName}.txt', or -NAME '{HSA Name}'}

.PARAMETER ALL

    Used to install all Hardware Support Applications in the subdirectories 
    below the script.

    -OFFLINE -ALL

.PARAMETER FILE

    Used to install a list of Hardware Support Applications.  The text file 
    should be formatted with one Hardware Support Application name per line.  
    The file should reside in the same folder as the Install-HSA.ps1 script.  
    The names can be found using the -LIST parameter.  

    -OFFLINE -FILE '{FileName}.txt'

.PARAMETER NAME

    Used to install one Hardware Support Application.  The name can be found 
    using the -LIST parameter.

    -OFFLINE -NAME '{HSA Name}'

.PARAMETER NOSMSTS

    Used when running in WinPE, but not in a Microsoft.SMS.TSEnvironment, 
    provided by Microsoft Endpoint Manager (MEM), System Center 
    Configuration Manager (SCCM), or Microsoft Deployment Toolkit (MDT).

    Call this parameter with the drive letter where the Windows partition has 
    the Windows folder already installed.

    -NOSMSTS '{Drive Letter}:'

.PARAMETER DEBUGINFORMATION

    Use to turn on Transcript logging and full logging of DISM Commands.  The 
    transcript file can be found at C:\Windows\Logs.  There will be a separate  
    log file from each DISM command generated at C:\Windows\Logs\DISM.

    -DEBUGINFORMATION

.EXAMPLE

    .\Install-HSA.ps1 -LIST

.EXAMPLE

    .\Install-HSA.ps1 -LIST -EXPORT

.EXAMPLE

    .\Install-HSA.ps1 -OFFLINE -NAME 'Lenovo Pen Settings'

.EXAMPLE

    .\Install-HSA.ps1 -OFFLINE -FILE 'List.txt'

.EXAMPLE

    .\Install-HSA.ps1 -OFFLINE -NOSMSTS 'D:' -ALL

.EXAMPLE

    .\Install-HSA.ps1 -OFFLINE -NOSMSTS 'D:' -ALL -DEBUGINFORMATION

.NOTES

    Return Code 1 : Both the -LIST and -OFFLINE commands were used.  Only one 
                    of these two parameters can be used at a time.
    Return Code 2 : More than one -ALL, -FILE, or -NAME were used.  Only one 
                    of these three parameters can be used at a time.
    Return Code 3 : No *_HSA_Manifest.json files were found in the 
                    subdirectories.
    Return Code 4 : When using the -FILE parameter, the file name was not 
                    found in the directory where the script resides.

#>

<#
Version history

- 1.03: Remove incompatible characters from the log file name.

- 1.02: Initial Release

#>

#######################
#  SCRIPT PARAMETERS  #
#######################
[CmdletBinding(DefaultParameterSetName = 'GetList')]
Param(
    [Parameter(ParameterSetName = 'GetList')]
    [switch]$List,
    [Parameter(ParameterSetName = 'GetList')]
    [switch]$Export,
    [Parameter(ParameterSetName = 'InstallOffline')]
    [switch]$Offline,
    [Parameter(ParameterSetName = 'InstallOffline')]
    [ValidateNotNullOrEmpty()]
    [string]$Name,
    [Parameter(ParameterSetName = 'InstallOffline')]
    [ValidateNotNullOrEmpty()]
    [string]$File,
    [Parameter(ParameterSetName = 'InstallOffline')]
    [switch]$All,
    [Parameter(ParameterSetName = 'InstallOffline')]
    [ValidateNotNullOrEmpty()]
    [string]$NoSMSTS,
    [Parameter(ParameterSetName = 'GetList')]
    [Parameter(ParameterSetName = 'InstallOffline')]
    [switch]$DebugInformation
)
###############
#  FUNCTIONS  #
###############
#Install-HSA
Function Install-HSA
{
    [CmdletBinding()]
    param (
        [PSCustomObject]$HSAPackage,
        [String]$HSAName
    )
    $OutDep = $Null
    If((($HSAName -contains $HSAPackage.hsa) -and ($Null -ne $HSAName))-or $All)
    {
        ForEach($Dep in $HSAPackage.Dependencies)
        {
            $OutDep += " /DependencyPackagePath:`"$($HSAPackage.JSONPath)\$($Dep)`""
        }
        $DISMLog = ""
        If($DebugInformation)
        {
            If(!(Test-Path -Path "$($LogPath)\DISM"))
            {
                New-Item -Path "$($LogPath)\DISM" -ItemType Directory
            }
            $HsaLogName = $($HSAPackage.hsa) -replace '\<|\>|:|"|/|\\|\||\?|\*', '_'
            $DISMLog = " /LogLevel:4 /LogPath:`"$LogPath\DISM\$HsaLogName.log`""
        }
        $DISMArgs = "/Add-ProvisionedAppxPackage /PackagePath:`"$($HSAPackage.JSONPath)\$($HSAPackage.appx)`" /LicensePath:`"$($HSAPackage.JSONPath)\$($HSAPackage.license)`"$($OutDep) /Region:`"All`"$DISMLog"
        Write-Host "Offline DISM - $($HSAPackage.hsa)"
        If($NoSMSTSPresent)
        {
            Write-Host "Using string data from NoSMSTS parameter to define the root drive letter for the DISM /Image parameter."
        }
        $DISMArgs = "/Image:$($Drive)\ $($DISMArgs)"
        Write-Host "$env:windir\system32\Dism.exe $DISMArgs"
        Start-Process -FilePath "$env:windir\system32\Dism.exe" -ArgumentList $DISMArgs -Wait -WindowStyle Hidden
    }
}
##################
#  SCRIPT SETUP  #
##################
$FilePresent = $false
$NamePresent = $false
$NoSMSTSPresent = $false
If(($PSBoundParameters.ContainsKey('List') -and $PSBoundParameters.ContainsKey('Offline')))
{
    Write-Host "Use just one from the following list of parameters: -List or -Offline.  Review the script usage information for using these parameters."
    Return 1
}
If($Offline)
{
    If(($PSBoundParameters.ContainsKey('All') -and $PSBoundParameters.ContainsKey('File') -and $PSBoundParameters.ContainsKey('Name')) -or ($PSBoundParameters.ContainsKey('All') -and $PSBoundParameters.ContainsKey('File')) -or ($PSBoundParameters.ContainsKey('All') -and $PSBoundParameters.ContainsKey('Name')) -or ($PSBoundParameters.ContainsKey('File') -and $PSBoundParameters.ContainsKey('Name')) -or ((!($PSBoundParameters.ContainsKey('All')) -and (!($PSBoundParameters.ContainsKey('File'))) -and (!($PSBoundParameters.ContainsKey('Name'))))))
    {
        Write-Host "Use just one from the following list of parameters: -All, -Name, or -File.  Review the script usage information for using these parameters."
        Return 2
    }
    ElseIf($PSBoundParameters.ContainsKey('File'))
    {
        $FilePresent = $true
    }
    ElseIf($PSBoundParameters.ContainsKey('Name'))
    {
        $NamePresent = $true
    }
}
If($PSBoundParameters.ContainsKey('NoSMSTS'))
{
    $NoSMSTSPresent = $true
}
#Setup Vars
$ScriptDir = Split-Path $Script:MyInvocation.MyCommand.Path
If(($Offline) -and (!($NoSMSTSPresent)))
{
    $TSenv = New-Object -COMObject Microsoft.SMS.TSEnvironment
    If($TSEnv.value("OSDTargetSystemDrive") -ne "")
    {
        $Drive = "$($TSEnv.value("OSDTargetSystemDrive"))"
    }
    Else
    {
        $Drive = "$($TSEnv.value("OSDisk"))"
    }
}
ElseIf(($Offline) -and ($NoSMSTSPresent))
{
    $Drive = $NoSMSTS
}
ElseIf($List)
{
    $Drive = $env:SystemDrive
}
$LogDate = Get-Date -Format yyyyMMddHHmmss
If($DebugInformation)
{
    $LogPath = "$($Drive)\Windows\Logs"
    $LogFile = "$LogPath\$($myInvocation.MyCommand)_$($LogDate).log"

######################
#  START TRANSCRIPT  #
######################
    Start-Transcript $LogFile -Append -NoClobber
    Write-Host "Debug enabled"
}
##########
#  MAIN  #
##########
$MFJs = Get-ChildItem -path $ScriptDir -Recurse -File -Include "*_manifest.json"
If($Null -eq $MFJs)
{
    Write-Host "No HSA_Manifest.JSON files found in the subfolder structure."
    Return 3
}
Else
{
    $HSAPackages = @()
    ForEach($MFJ in $MFJs)
    {
        $MFJData = Get-Content -Path "$($MFJ.FullName)" | ConvertFrom-Json
        $HSAPackages += New-Object PSObject -Property @{'JSONPath' = $MFJ.DirectoryName; 'HSA' = $MFJData.HSA; 'Appx' = $MFJData.Appx; 'License' = $MFJData.License; 'Dependencies' = $MFJData.Dependencies}
    }
}
If($List -or $PSBoundParameters.Count -eq 0 -or ($DebugInformation -and $PSBoundParameters.Count -eq 1))
{
    If($PSBoundParameters.ContainsKey('Export'))
    {
        ForEach($Package in $HSAPackages)
        {
            $Package.hsa | Out-File "$ScriptDir\Export_$LogDate.txt" -Append -Noclobber
        }
    }
    Else
    {
        ForEach($Package in $HSAPackages)
        {
            Write-Host $Package.hsa
        }
    }
}
If($Offline)
{
    If($All)
    {
        Write-Host "Installing all HSAs found in the folder structure."
    }
    ElseIf($NamePresent)
    {
        Write-Host "Installing the $Name HSA."
    }
    ElseIf($FilePresent)
    {
        Write-Host "Reading the list of HSAs from $File."
        If(!(Test-Path -Path "$ScriptDir\$File"))
        {
            Write-Host "File: $File not found in $ScriptDir"
            Return 4
        }
    }
    ForEach($Package in $HSAPackages)
    {
        If($All)
        {
            Install-HSA -HSAPackage $Package
        }
        ElseIf($FilePresent -or $NamePresent)
        { 
            If($FilePresent)
            {
                $InstallFileArray = @()
                $InstallFileArray = Get-Content -Path "$ScriptDir\$File"
                ForEach($InstallFile in $InstallFileArray)
                {
                    Install-HSA -HSAPackage $Package -HSAName $InstallFile
                }
            }
            If($NamePresent)
            {
                Install-HSA -HSAPackage $Package -HSAName $Name
            }
        }
        $InstallFileArray = $Null
    }
}
If($DebugInformation)
{
    #####################
    #  STOP TRANSCRIPT  #
    #####################
    Stop-Transcript
}

```

* **List Functionality** – Used to list the names of all Hardware Support Applications in the subdirectories.  Returns the list of all Hardware Support Applications names to the screen.

* **Export** – Used to list the names of all Hardware Support Applications in the subdirectories.  Returns the list of all Hardware Support Applications names to the Export_{Date}{Time}.log file in the same directory from which the script was executed.

* **Offline Install Functionality** – Used to install the Hardware Support Applications.
  * **All** – Used to install all Hardware Support Applications in the subdirectories below the script.
  * **File** – Used to install a list of Hardware Support Applications.  The text file should be formatted with one Hardware Support Application name per line.  The file should reside in the same folder as the Install-HSA.ps1 script.  The names can be found using the -LIST parameter
  * **Name** – Used to install one Hardware Support Application.  The name can be found using the -LIST

* **NoSMSTS** – used when running in winPE, but not in a Microsoft.SMS.TSEnvironment, provided by Microsoft Endpoint Manager (MEM) / System Center Configuration Manager (SCCM) or Microsoft Deployment Toolkit (MDT).
  * Call this parameter with the drive letter and colon (:) where the Windows partition has the Windows folder already installed.

* **DebugInformation** – Used to turn on transcript logging and full logging of DISM commands.  The transcript file can be found at C:\Windows\Logs.  There will be a separate log file from each DISM command generated at C:\Windows\Logs\DISM.

For MEM/SCCM and MDT implementations, we are providing the following guidance for the items needed to successfully deploying Lenovo devices. Each implementation has guidance on WinPE Optional Components Requirements, the package to convey the content to the device, and the Task Sequence task information to perform the install.

## MEM/SCCM Implementation

### Windows PE Optional Components Requirements

![PE Components](https://cdrt.github.io/mk_blog/img/2020/hsa_winpe_oc.png)

* Microsoft .NET (WinPE-NetFx)
* Windows PowerShell (WinPE-PowerShell)
* Windows PowerShell (WinPE-DismCmdlets)

### Packaging

![HSA folder](https://cdrt.github.io/mk_blog/img/2020/hsa_folder_structure_w_script.png)

1. Download and extract the required HSA package from Lenovo's website.
2. Copy the **Install-HSA.ps1** script to the directory where the HSA package was extracted.
3. Create a legacy package with the source files pointed to the directory where the HSA pack is extracted.  There is no need to create a program associated with this legacy package.
4. Distribute to your environment.

!!! note
    If you are going to install more than one HSA, but not all, and are going to use the -FILE parameter, be sure to include the text file with the list of HSA Names in the same directory as the script file.

### Task Sequence

![Run PowerShell Script](https://cdrt.github.io/mk_blog/img/2020/hsa_ts_ps_task_1.png)

![Options](https://cdrt.github.io/mk_blog/img/2020/hsa_ts_ps_task_2.png)

1. After the Apply Operating System task, but before the Setup Windows and Configuration Manager task, add a **Run PowerShell Script** task.
1. On the Properties tab, choose the **Select a package with a PowerShell script** option.  Click the Browse button to locate and select the package created above.
1. In the Script name textbox, enter **Install-HSA.ps1**.
1. In the Parameters textbox, enter the parameters required.  Ex. -Offline -All -DebugInformation.
1. On the Options tab, if needed, add the **conditional statement** and any **WMI queries** to target by model.

## MDT Implementation

### WinPE Features Requirements

![PE components](https://cdrt.github.io/mk_blog/img/2020/hsa_winpe.png)

* .NET Framework
* Windows PowerShell
* DISM Cmdlets

### Application

![HSA folder](https://cdrt.github.io/mk_blog/img/2020/hsa_folder.png)

1. Download and extract the required HSA package from Lenovo's website.
1. Copy the **Install-HSA.ps1** script to the directory where the HSA package was extracted.
1. Create an application in MDT.  We will use this to neatly store the files in MDT.  When the wizard asks for executable, you can put anything, as we will not be using the application functionality to install the HSA.

!!! note
    If you are going to install more than one HSA, but not all, and are going to use the -FILE parameter, be sure to include the text file with the list of HSA Names in the same directory as the script file.

### MDT Task Sequence

![Run command line](https://cdrt.github.io/mk_blog/img/2020/hsa_mdt_ts1.png)

![Options](https://cdrt.github.io/mk_blog/img/2020/hsa_mdt_ts2.png)

1. In the Task Sequence, after the Apply Operating System task in the Install phase, but before the Restart Computer task in the PostInstall phase, add a Run Command Line task and give it a name.
1. In the command line, enter the following command:
```cmd
powershell.exe -WindowStyle Hidden -ExecutionPolicy Bypass -Command "Copy-Item '%DeployRoot%\Applications\{ApplicationFolderNameHere}' -Destination %OSDisk%\OSDTemp\ -Recurse; %OSDisk%\OSDTemp\Install-HSA.ps1 -OFFLINE -ALL -DEBUGINFORMATION; Remove-Item %OSDisk%\OSDTemp -Recurse -Force"
```
The command above will copy all files and folders from the Application folder defined, execute the Install-HSA.ps1 script, and remove the content from the %OSDisk% drive.
1. Be sure to change {ApplicationFolderNameHere} to the actual folder name of the Application.
1. On the **Options** tab, if needed, add the **conditional statement** and any **WMI queries** to target by model.

## Noteworthy

* **Not Applicable HSAs** - Even though an HSA is in a Lenovo supplied HSA pack, that does not necessarily mean it will apply to every build of a model.  For example:
  * On the ThinkPad T15 Gen 1, not all builds of this model will have an Nvidia Graphics Card, so the Nvidia Graphics HSA may or may not be needed.
  * Many models have an option to utilize Intel Optane Storage.  The HSA is only needed if the device contains the Optane disk drive.
* **HSAs not installing as SYSTEM account** - In an MEM/SCCM Task Sequence, after the Setup Windows and Configuration Manager task, the task sequence runs as the SYSTEM account.  Many HSAs will not install under the SYSTEM account, so we recommend installing in WinPE.
