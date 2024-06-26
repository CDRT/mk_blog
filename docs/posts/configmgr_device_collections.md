---
date: 2022-01-25
authors:
    - Phil
categories:
    - "2022"
title: Automatically Create ConfigMgr Device Collections for Lenovo Products
---

This is a PowerShell based solution that will query the ConfigMgr database for Lenovo branded products and create a Device Collection based off the friendly name, i.e. **ThinkPad T14 Gen 2**.  Each device will then be moved to its respective collection.
<!-- more -->
## PREREQUISITES

In order for the friendly name to be queried, the **Win32_ComputerSystemProduct** WMI class will need to be added/enabled to the ConfigMgr hardware inventory.  This needs to be performed in the Default Client Settings in the **Client Settings** node under the **Administration** workspace.  For a step by step, refer to the this MS [**doc**](https://docs.microsoft.com/en-us/mem/configmgr/core/clients/manage/inventory/extend-hardware-inventory).

## SOLUTION

Script can be downloaded from my [GitHub](https://github.com/philjorgensen/ConfigMgr/blob/main/Device%20Collections/Create-LenovoDeviceCollections.ps1).

Run in an elevated PowerShell on a system that has the ConfigMgr console installed.  This also assumes you have the necessary permissions to create Device Collections in your environment.

``` powershell title="Create-LenovoDeviceCollections.ps1"
<#
.SYNOPSIS
    Creates ConfigMgr Device Collections for Lenovo systems
.DESCRIPTION
    Script that checks the ConfigMgr database for Lenovo branded systems and creates a Device Collection based off the friendly name.

    Example: ThinkPad T14 Gen 2
.PARAMETER SiteServer
    Fully qualified domain name of Site Server
.EXAMPLE
    .\Create-LenovoDeviceCollections.ps1 -SiteServer cm01.domain.com
.NOTES
    Author: Philip Jorgensen
    Created: 12/16/2021

    Run script from Site Server or system with the ConfigMgr console installed.
#>

[CmdletBinding()]
param (
    [Parameter(Mandatory = $true,
        HelpMessage = "FQDN of Site Server",
        ValueFromPipeline = $false)
    ]
    [ValidateNotNullOrEmpty()]
    [OutputType([String])]$SiteServer
)

$ErrorActionPreference = "SilentlyContinue"

#Import ConfigMgr PS Module
if (!(Get-Module -Name ConfigurationManager)) {
    Import-Module $env:SMS_ADMIN_UI_PATH.Replace("bin\i386", "bin\ConfigurationManager.psd1")
}
    
#Connect to ConfigMgr Site 
Write-Host "Connecting to $SiteServer..." -ForegroundColor Yellow
$SiteCode = $(Get-CimInstance -ComputerName $SiteServer -Namespace "root\SMS" -Class "SMS_ProviderLocation").Sitecode

if (!(Get-PSDrive -Name $Sitecode)) {
    New-PSDrive -Name $Sitecode -PSProvider CMSite -Root $SiteServer -Description "Primary Site Server"
}

Set-Location -Path $Sitecode":\DeviceCollection"

<# 
    Queries system by Lenovo version.
    NOTE: The Computer System Product (Win32_ComputerSystemProduct) hardware inventory class with Vendor property must be enabled.
#>

$Subfolder = "Lenovo"  
$Models = Get-CimInstance -ComputerName $SiteServer -Namespace "root\SMS\site_$($Sitecode)" -Query "Select * From SMS_G_System_COMPUTER_SYSTEM_PRODUCT Where Vendor = 'LENOVO'" | Select-Object -Property Vendor, Version | Sort-Object -Unique -Property Vendor, Version

if ($Models.Count -ne '0') {
    if (!(Test-Path -Path $Subfolder)) {
        New-Item -Name $Subfolder
    }

    $i = 1
    foreach ($Model in $Models) {
        $sModel = $Model.Version
        $iModel = $Models.Count
        Write-Progress "Adding Lenovo devices to named device collections." -Status "Updating the $sModel collection. $i of $iModel" `
            -PercentComplete ($i++ / $Models.count * 100)

        <# 
            Creates collection based on version of Lenovo system and sets schedule to update collection weekly.
            Adjust schedule as desired.
        #>
        #Adjust limiting collection as desired.
        $Schedule = New-CMSchedule -DayOfWeek Monday -Start "2/17/2016 03:00:00 AM"
        $ModelVersion = $Model.Version

        if (!(Get-CMDeviceCollection -Name $ModelVersion)) {
            $NewCollection = New-CMDeviceCollection -Name "$ModelVersion" -LimitingCollectionName 'All Systems' -RefreshType Periodic -RefreshSchedule $Schedule
            $LenovoCollection = Get-CMDeviceCollection -Name "$ModelVersion"
            Move-CMObject -InputObject $LenovoCollection -FolderPath "$Subfolder"
        }
        $CollectionQuery = "Select *  From  SMS_R_System inner join SMS_G_System_COMPUTER_SYSTEM_PRODUCT on SMS_G_System_COMPUTER_SYSTEM_PRODUCT.ResourceId = SMS_R_System.ResourceId where SMS_G_System_COMPUTER_SYSTEM_PRODUCT.Version = ""$ModelVersion"""
        if (!(Get-CMDeviceCollectionQueryMembershipRule -CollectionName $ModelVersion -RuleName $ModelVersion)) {
            Add-CMDeviceCollectionQueryMembershipRule -CollectionName "$ModelVersion" -QueryExpression $CollectionQuery -RuleName $ModelVersion
        }
    }

    # Disconnect from Site
    Write-Host "Device Collections updated." -ForegroundColor Yellow
    Write-Host "Disconnecting from Site..." -ForegroundColor Yellow
    Set-Location -Path $env:SMS_ADMIN_UI_PATH
    Remove-PSDrive -Name $Sitecode
    Set-Location -Path $env:HOMEPATH    
}

Else {
    Write-Host "No Lenovo Models detected in the CM database..."
}
```

## RESULT

This was tested in a lab with very few devices so this may or may not take some time to finish depending on the amount of endpoints in your environment.  In the end, you should see a **Lenovo** subfolder containing all of your Lenovo Device Collections.  Hopefully this helps with organization!

![Device Collections](https://cdrt.github.io/mk_blog/img/2022/configmgr_device_collections/image1.jpg)
