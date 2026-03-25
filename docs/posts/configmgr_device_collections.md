---
date:
    created: 2022-01-25
    updated: 2026-03-25
authors:
    - Phil
categories:
    - "2022"
title: Automatically Create ConfigMgr Device Collections for Lenovo Products
---

This is a PowerShell based solution that will query the ConfigMgr database for Lenovo branded products and create a Device Collection based off the friendly name, i.e. **ThinkPad X1 Carbon Gen 13**. Each device will then be moved to its respective collection.
<!-- more -->
## Prerequisites

In order for the friendly name to be queried, the **Win32_ComputerSystemProduct** WMI class will need to be added/enabled to the ConfigMgr hardware inventory. This needs to be performed in the Default Client Settings in the **Client Settings** node under the **Administration** workspace. For a step-by-step guide, refer to this [**Microsoft doc**](https://learn.microsoft.com/intune/configmgr/core/clients/manage/inventory/extend-hardware-inventory).

## Solution

Script can be downloaded from my [GitHub](https://github.com/philjorgensen/ConfigMgr/tree/main/Device%20Collections).

Run in an elevated PowerShell session on a system that has the Configuration Manager console installed. This also assumes you have the necessary permissions to create Device Collections in your environment.

``` powershell title="New-LenovoDeviceCollections.ps1"
<#
.SYNOPSIS
    Creates ConfigMgr Device Collections for Lenovo systems

.DESCRIPTION
    Script that checks the ConfigMgr database for Lenovo branded systems and creates a Device Collection based off the friendly name.

    Example: ThinkPad X1 Carbon Gen 13

.PARAMETER SiteServer
    Fully qualified domain name of Site Server

.EXAMPLE
    .\New-LenovoDeviceCollections-Updated.ps1 -SiteServer cm01.domain.com

.NOTES
    Author:     Philip Jorgensen
    Created:    2021-12-16
    Updated:    2026-02-26
    Filename:   New-LenovoDeviceCollections.ps1

    Version history:
    1.0 - Initial script development and testing
    2.0 - Refactored code for improved error handling, added progress indicators, and enhanced collection management logic

    Run script from Site Server or system with the ConfigMgr console installed.
#>

[CmdletBinding()]
[OutputType([String])]
param (
    [Parameter(Mandatory = $true,
        HelpMessage = "FQDN of Site Server",
        ValueFromPipeline = $false)]
    [ValidateNotNullOrEmpty()]
    $SiteServer
)

$ErrorActionPreference = "Stop"

function Connect-SCCMSite
{
    param (
        [string]$SiteServer
    )

    try
    {
        Write-Host "Importing ConfigMgr module" -ForegroundColor Yellow
        Import-Module $env:SMS_ADMIN_UI_PATH.Replace("bin\i386", "bin\ConfigurationManager.psd1") -Force
    }
    catch
    {
        throw "Failed to import ConfigMgr module"
    }

    try
    {
        # Retrieve Site Code from SCCM server
        $SiteCode = Get-CimInstance -ComputerName $SiteServer -Namespace root/SMS -ClassName SMS_ProviderLocation -ErrorAction Stop |
            Select-Object -ExpandProperty SiteCode -First 1

        if (-not $SiteCode)
        {
            throw "Failed to retrieve Site Code from $SiteServer."
        }

        # Display progress while connecting
        Write-Progress -Activity "Connecting to Site Server" -Status "Creating PSDrive for SCCM"

        # Create PSDrive for SCCM
        if (-not (Get-PSDrive -PSProvider CMSite -Name $SiteCode -ErrorAction SilentlyContinue))
        {
            New-PSDrive -Name $SiteCode -PSProvider CMSite -Root $SiteServer -Description "Primary Site Server" -Scope Global -ErrorAction Stop | Out-Null
        }

        # Set location to SCCM PSDrive
        Set-Location -Path "$SiteCode`:"

        Write-Host "Successfully connected to SCCM Site Server: $SiteServer ($SiteCode`:)" -ForegroundColor Green
        return $SiteCode
    }
    catch
    {
        throw "Error connecting to SCCM Site Server: $_"
    }
}

function New-LenovoDeviceCollections
{
    param (
        [string]$SiteServer,
        [string]$SiteCode
    )
    $Vendor = "Lenovo"
    $Subfolder = Get-CMFolder -ObjectTypeName SMS_Collection_Device -Name $Vendor -ErrorAction SilentlyContinue
    $Models = Get-CimInstance -ComputerName $SiteServer -Namespace "root\SMS\site_$($SiteCode)" -Query "Select * From SMS_G_System_COMPUTER_SYSTEM_PRODUCT Where Vendor = 'LENOVO'" | Select-Object -Property Version | Sort-Object -Unique -Property Version

    if ($Models.Count -ne 0)
    {
        if ($null -eq $Subfolder)
        {
            $Subfolder = New-CMFolder -Name $Vendor -ParentFolderPath 'DeviceCollection' -ErrorAction Stop | Out-Null
        }

        $i = 1
        $iModel = $Models.Count
        $Days = @('Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday')
        foreach ($Model in $Models)
        {
            $ModelVersion = $Model.Version
            Write-Progress "Adding devices to named device collections." -Status "Updating the $ModelVersion collection. $i of $iModel" -PercentComplete ($i++ / $iModel * 100)

            # Generate a unique staggered schedule per collection
            $RandomDay = Get-Random -InputObject $Days
            $RandomHour = Get-Random -Minimum 0 -Maximum 24
            $RandomMinute = Get-Random -InputObject @(0, 15, 30, 45)
            $Schedule = New-CMSchedule -DayOfWeek $RandomDay -Start (Get-Date -Hour $RandomHour -Minute $RandomMinute -Second 0)

            try
            {
                if (-not (Get-CMDeviceCollection -Name $ModelVersion))
                {
                    Write-Host "Creating collection for $ModelVersion" -ForegroundColor Cyan
                    $NewCollection = New-CMDeviceCollection -Name "$ModelVersion" -LimitingCollectionName 'All Systems' -RefreshType Periodic -RefreshSchedule $Schedule

                    Move-CMObject -InputObject $NewCollection -FolderPath $SiteCode":\DeviceCollection\$Vendor"

                }
                $CollectionQuery = "Select * From SMS_R_System inner join SMS_G_System_COMPUTER_SYSTEM_PRODUCT on SMS_G_System_COMPUTER_SYSTEM_PRODUCT.ResourceId = SMS_R_System.ResourceId where SMS_G_System_COMPUTER_SYSTEM_PRODUCT.Version = '$ModelVersion'"
                if (-not (Get-CMDeviceCollectionQueryMembershipRule -CollectionName $ModelVersion -RuleName $ModelVersion))
                {
                    Add-CMDeviceCollectionQueryMembershipRule -CollectionName "$ModelVersion" -QueryExpression $CollectionQuery -RuleName $ModelVersion
                }
            }
            catch
            {
                Write-Warning "Failed to process collection for '$ModelVersion': $_"
            }
        }
    }
    else
    {
        Write-Host "No Lenovo Models detected in the CM database..." -ForegroundColor Red
    }
}

function Disconnect-ConfigMgrSite
{
    param (
        [string]$SiteCode
    )
    Write-Host "Device Collections updated." -ForegroundColor Green
    Write-Host "Disconnecting from Site..." -ForegroundColor Yellow
    Set-Location -Path $env:USERPROFILE
    Remove-PSDrive -Name $SiteCode -ErrorAction SilentlyContinue
}

# Main
$SiteCode = Connect-SCCMSite -SiteServer $SiteServer
if ($SiteCode)
{
    New-LenovoDeviceCollections -SiteServer $SiteServer -SiteCode $SiteCode
    Disconnect-ConfigMgrSite -SiteCode $SiteCode
}
```

## Result

This was tested in a lab with very few devices. Execution time will vary depending on the number of devices in your environment.

![Device Collections](https://cdrt.github.io/mk_blog/img/2022/configmgr_device_collections/image1.jpg)

![Device Collections](https://cdrt.github.io/mk_blog/img/2022/configmgr_device_collections/image2.jpg)

In the end, you should see a **Lenovo** subfolder under **Device Collections** with all new model-based collections. The script also randomizes the full update schedule for each collection so they're not evaluating at the same time. Hopefully this helps with organization!

![Device Collections](https://cdrt.github.io/mk_blog/img/2022/configmgr_device_collections/image3.jpg)
