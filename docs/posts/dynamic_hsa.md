---
date:
    created: 2020-07-29
    updated: 2024-06-24
authors:
    - Phil
categories:
    - "2020"
title: Dynamically Install Hardware Support Apps with ConfigMgr
---

This article is intended to provide another solution to install HSA packs for your Think product in a ConfigMgr Task Sequence.

For an in-depth overview of these HSA packs and installation script, refer to this [**article**](https://blog.lenovocdrt.com/2020/hsa-1.md).  

To make this process a bit easier and to reduce the number of steps in your Task Sequence, this solution goes hand in hand with an older post: [**Dynamically Updating BIOS**](https://blog.lenovocdrt.com/2017/dynamic_bios_update.md).
<!-- more -->
## Workflow

We can achieve this in essentially 3 steps in a Task Sequence.

- Set the correct Package to download for the applicable model (PowerShell script)
- Download Package to a custom path on the client
- Installation using the Install-HSA.ps1 script found [**here**](https://blog.lenovocdrt.com/2020/hsa-1.md)

---

### Download HSA Pack (Manual Way)

First, you'll need to download/extract the contents of the HSA pack to a desired location and place the **Install-HSA.ps1** file in the top level directory.

![HSA pack](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image1.jpg)

#### Create Legacy Package(s)

In the ConfigMgr console, create a Package (No Program) and enter the following details

- **Name** - Friendly name of the system.  For example, **ThinkPad X13 Yoga Gen 1**
  - This can be found in the deployment [**recipe card**](https://download.lenovo.com/cdrt/ddrc/RecipeCardWeb.html) for the model
- **Version** - This is optional, but I entered the HSA pack version, which can be found in the ReadMe or typically in the pack's file name.
- **Description/Comment** - This is the first 4 characters of the Machine Type Model. For Example, **21D6,21D7**
  - Also found in the deployment recipe card

![Package details](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image2.jpg)

- **MIF File Name** - HSA
- **MIF Name** - OS version. **win10** or **win11**
- **MIF Version** - Windows Build. For example **22H2** or **22H2_23H2** (if deploying to Windows 10/11 23H2). **Note:** This is not to be used as a parameter value in the task sequence.

!!! info ""
    If there are no 23H2 HSA packs available, use the 22H2 packs.

![MIF Matching](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image3.jpg)

### Download HSA Pack (Automated Way)

We have added a new node in the Think Deploy [catalog](https://download.lenovo.com/cdrt/td/catalogv2.xml) for HSAs. With this, we can automate the download, extraction, and creation of ConfigMgr packages with the PowerShell script **New-LnvHsaConfigMgrPackage.ps1**.

![Select HSA pack](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image4.jpg)

You can select a single pack to download or ctrl + click for multiple. The packs will download and extract to the temp directory, moved to the share you specify, and finally ConfigMgr Packages are created.

![Script completion](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image5.jpg)

!!! info ""
    You'll need to distribute the content to your Distribution Points

The script can be found on my GitHub here

<https://github.com/philjorgensen/ConfigMgr/tree/main/Packages>

### Generating the Packages XML

If you don't already have a Package containing your Scripts, create another one for this purpose.

Referencing the Dynamic BIOS Update post, you'll need to generate an XML containing your Packages. This XML will contain the necessary data in order to match the HSA Package to your Think product. To generate the XML, run the below code

```powershell
# Connect to ConfigMgr Site 

$SiteCode = $(Get-WmiObject -ComputerName "$ENV:COMPUTERNAME" -Namespace "root\SMS" -Class "SMS_ProviderLocation").SiteCode

# Get Package data and export XML
Get-WmiObject -Class SMS_Package -Namespace root\SMS\Site_$SiteCode | Select-Object Pkgsourcepath, Description, Manufacturer, MifFileName, MifName, MIFVersion, Name, PackageID, ShareName, Version | Sort-Object -Property Name | Export-Clixml -Path '_Packages.xml' -Force 
```

If you open the XML, the contents should be similar to this

![XML](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image6.jpg)

Copy this XML to your Scripts folder.  Along with the XML, another piece to the puzzle is needed to be able to grab the correct HSA Package during the Task Sequence.  The below PowerShell script (**Get-DynamicHsaPackages.ps1**) will look at the Packages.xml, match the Name/MTM to it's corresponding HSA Package, and leverage the **OSDDownloadDownloadPackages** override variable in the Download Package Content step.  This script needs to be saved in your Scripts folder as well.

```powershell
[CmdletBinding()]
param (
    [Parameter(ValueFromPipelineByPropertyName, Position = 0)]
    [string] $MatchProperty = 'Description',

    [Parameter(ValueFromPipelineByPropertyName, Position = 1)]
    [string] $MachineType = (Get-CimInstance -Namespace root/CIMV2 -ClassName Win32_ComputerSystemProduct).Name.Substring(0, 4).Trim(),

    [Parameter(ValueFromPipelineByPropertyName, Position = 2)]
    [string] $PackageXMLLibrary = ".\_Packages.xml",

    [Parameter(ValueFromPipelineByPropertyName, Position = 3)]
    [ValidateSet("win10", "win11")]
    [string] $WindowsVersion = "",

    [Parameter(ValueFromPipelineByPropertyName, Position = 4)]
    [ValidateSet("1709", "1803", "1809", "1903", "1909", "2004", "20H2", "21H1", "21H2", "22H2", "23H2", "24H2")]
    [string] $WindowsBuild = ""
)

# Initialize task sequence environment if available
$tsenvInitialized = $false
try
{
    $tsenv = New-Object -ComObject Microsoft.SMS.TSEnvironment
    $tsenvInitialized = $true
}
catch
{
    Write-Host 'Not executing in a task sequence'
}

# Find the PackageID based on the criteria
try
{
    $PackageID = (Import-Clixml -Path $PackageXMLLibrary | Where-Object { 
            $_.$MatchProperty.Split(',').Contains($MachineType) -and
            $_.MifFileName -eq "HSA" -and
            $_.MifName -eq $WindowsVersion -and
            $_.MifVersion -match $WindowsBuild }).PackageID
}
catch
{
    Write-Error "Failed to find the PackageID. Error: $_"
    exit 1
}

# Output the PackageID
Write-Output("$PackageID for $MachineType will be downloaded")

# Set the task sequence variable if initialized
if ($tsenvInitialized)
{
    try
    {
        $tsenv.Value('OSDDownloadDownloadPackages') = $PackageID
    }
    catch
    {
        Write-Error "Failed to set task sequence variable. Error: $_"
    }
}
```

### Putting It All Together

!!! info ""
    Confirm the **Windows Powershell (WinPE-PowerShell)** optional component is added to your boot image

In my testing, I created a Child Task Sequence containing everything above and have added to my main Task Sequence. Here's what it would look like

**Run PowerShell Script**: Select the Scripts Package containing:

- _.Packages XML
- Get-DynamicHsaPackages.ps1

**Script Name**: Get-DynamicHsaPackages.ps1

**Parameters**: -WindowsVersion 'win11' -WindowsBuild '23H2'

- If you're deploying Windows 11 23H2, these are the parameters you'll set.

**PowerShell execution policy**: Set to Bypass

![Task sequence](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image7.jpg)

**Download Package Content**: This step will eventually get overridden due to the OSDDownloadDownloadPackages variable being set in the Get-DynamicHsaPackages script. So create an empty Package and add it here.

**Custom path**: This is where the HSA Package will be downloaded to on the client. Here, I'm using the %_SMSTSMDataPath% (I have my drivers set to download here as well). On the client, this will resolve to C:\_SMSTaskSequence\HSAs\HSA_PackageID

![Task sequence](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image8.jpg)

**Run Command Line**: This step calls PowerShell to execute the Install-HSA.ps1 with parameters to install all HSAs offline (WinPE).

```cmd
powershell.exe -ExecutionPolicy Bypass -Command (%_SMSTSMDataPath%\HSAs\*\Install-HSAs.ps1 -Offline -All -DebugInformation)
```

![Task sequence](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image9.jpg)

This being a Child Task Sequence, I've added it to my main Task Sequence right after my Install Drivers step and before the Setup Windows and ConfigMgr Client step

![Child task sequence](https://cdrt.github.io/mk_blog/img/2020/dynamic_hsa/image10.jpg)
