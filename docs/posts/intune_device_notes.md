---
date:
    created: 2023-02-07
    updated: 2025-03-19
authors:
    - Phil
categories:
    - "2023"
title: Adding Model Friendly Name to Intune Device Notes
---

As of today, there's still a limitation within the Intune portal to easily find the friendly name of a Lenovo system, i.e. **ThinkPad T14 Gen 3**. Instead, you're left with the Machine Type Model (**21AH**).

I'm sure there's a small group of people, if any, that have memorized what every MTM translates to for its respective friendly name.

To make this a bit easier, and with the help of Damien Van Robaeys' [blog post](https://www.systanddeploy.com/2023/01/get-list-uptodate-of-all-lenovo-models.html), we can use the Graph API to populate the device notes property of an Intune device by matching the **Model** (aka MTM) to its friendly name.
<!-- more -->
The below code can be used to accomplish this. If there's currently a note set for a device, the friendly name will be appended on the next line of the note. You can also download from my GitHub [here](https://github.com/philjorgensen/Graph/blob/main/Set-DeviceNoteFriendlyName.ps1).

This was tested in a small tenant so results may vary as far as time to completion goes.

!!! info
    PowerShell 7 is recommended

## Graph PowerShell SDK cmdlets

This option uses the cmdlets from the [Graph PowerShell SDK](https://learn.microsoft.com/powershell/microsoftgraph/overview?view=graph-powershell-1.0).

```powershell
#requires -Module Microsoft.Graph.Authentication
#requires -Module Microsoft.Graph.DeviceManagement

Disconnect-Graph -ErrorAction SilentlyContinue
Connect-MgGraph -Scopes DeviceManagementManagedDevices.ReadWrite.All, Directory.Read.All

# No longer needed in Graph SDK V2
# Select-MgProfile -Name beta

# Filter for Lenovo devices
$managedDevices = Get-MgDeviceManagementManagedDevice -Filter "Manufacturer eq 'LENOVO'"

<#

Variables for MTM to Friendly Name
https://github.com/damienvanrobaeys/Lenovo_Models_Reference/blob/main/MTM_to_FriendlyName.ps1

#>

$URL = "https://download.lenovo.com/luc/bios.txt#"
$Get_Web_Content = (Invoke-WebRequest -Uri $URL).Content
$Models = $Get_Web_Content -split "`r`n"

foreach ($device in $managedDevices)
{

    $deviceNotes = (Get-MgDeviceManagementManagedDevice -ManagedDeviceId $device.Id -Property "Notes").Notes
    $Mtm = $device.Model.Substring(0, 4).Trim()
    [string]$FamilyName = $(foreach ($Model in $Models)
        {
            if ($Model.Contains($Mtm))
            {
                if ($Model.Contains("Type"))
                {
                    Write-Host $Model.Split("Type")[0]
                }
                else
                {
                    $Model.Split("=")[0]
                }
            }
        }) | Sort-Object -Unique

    if ([string]::IsNullOrEmpty($deviceNotes))
    {

        # Update Device notes
        Update-MgDeviceManagementManagedDevice -ManagedDeviceId $device.Id -Notes $FamilyName

    }
    elseif ($deviceNotes -notmatch $FamilyName)
    {

        $appendDeviceNote = $deviceNotes + "`n$FamilyName"
        Update-MgDeviceManagementManagedDevice -ManagedDeviceId $device.Id -Notes $appendDeviceNote
    }
}

<#

# Output the results
foreach ($device in $managedDevices) {
    $deviceNotes = (Get-MgDeviceManagementManagedDevice -ManagedDeviceId $device.Id -Property "Notes").Notes
    Write-Output -InputObject "$($device.DeviceName) is a $($deviceNotes)"
}

#>
```

## Invoke-MgGraphRequest

This other option makes direct calls to the Graph API using [Invoke-MgGraphRequest](https://learn.microsoft.com/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest?view=graph-powershell-1.0).

```powershell
#requires -Module Microsoft.Graph.Authentication

Connect-MgGraph -Scopes DeviceManagementManagedDevices.ReadWrite.All, Directory.Read.All

# Define constants
$endpoint = "https://graph.microsoft.com"
$version = "beta"
$resource = "deviceManagement/managedDevices"
$query = "?`$filter=manufacturer eq 'LENOVO'"
$query2 = "?`$select=notes"

# Query managed Lenovo devices via Microsoft Graph API
$devicesRequest = @{
    Uri    = "$($endpoint)/$($version)/$($resource)$($query)"
    Method = "GET"
}

try
{
    $managedDevices = (Invoke-MgGraphRequest @devicesRequest).value
}
catch
{
    Write-Error "Failed to retrieve managed devices: $_.Exception.Message"
    return
}

<#
Variables for MTM to Friendly Name
https://github.com/damienvanrobaeys/Lenovo_Models_Reference/blob/main/MTM_to_FriendlyName.ps1
#>
$URL = "https://download.lenovo.com/luc/bios.txt#"
$Get_Web_Content = (Invoke-WebRequest -Uri $URL).Content
$Models = $Get_Web_Content -split "`r`n"

foreach ($device in $managedDevices)
{
    $deviceDetailsRequest = @{
        Uri    = "$($endpoint)/$($version)/$($resource)/$($device.id)$($query2)"
        Method = "GET"
    }

    try
    {
        $deviceDetails = Invoke-MgGraphRequest @deviceDetailsRequest
    }
    catch
    {
        Write-Error "Failed to retrieve device details for device ID $($device.id): $_.Exception.Message"
        continue
    }

    $deviceNotes = $deviceDetails.notes
    $Mtm = if ($device.model.Length -ge 4) { $device.model.Substring(0, 4).Trim() } else { $device.model.Trim() }
    [string]$FamilyName = $(foreach ($Model in $Models)
        {
            if ($Model.Contains($Mtm))
            {
                if ($Model.Contains("Type"))
                {
                    $Model.Split("Type")[0]
                }
                else
                {
                    $Model.Split("=")[0]
                }
            }
        }) | Sort-Object -Unique

    $notesRequest = @{
        Uri    = "$($endpoint)/$($version)/$($resource)/$($device.id)"
        Method = "PATCH"
    }

    if ([string]::IsNullOrEmpty($deviceNotes))
    {
        # Update Device notes
        $notesRequest.Body = (@{ notes = $FamilyName } | ConvertTo-Json -Compress)
        try
        {
            Invoke-MgGraphRequest @notesRequest
            Write-Host "Updated notes for device ID $($device.id) to $($FamilyName)"
        }
        catch
        {
            Write-Error "Failed to update notes for device ID $($device.id): $_.Exception.Message"
        }
    }
    elseif ($deviceNotes -notmatch $FamilyName)
    {
        $appendDeviceNote = $deviceNotes + "`n$FamilyName"
        $notesRequest.Body = (@{ notes = $appendDeviceNote } | ConvertTo-Json -Compress)
        try
        {
            Invoke-MgGraphRequest @notesRequest
            Write-Host "Appended notes for device ID $($device.id) with $($FamilyName)"
        }
        catch
        {
            Write-Error "Failed to append notes for device ID $($device.id): $_.Exception.Message"
        }
    }
    else
    {
        Write-Host "No update needed for device ID $($device.id)"
    }
}
```
## Results

Once finished, check a device's notes in the portal to find it's friendly name.

![DeviceNote](https://cdrt.github.io/mk_blog/img/2023/intune_device_notes/image1.png)

If you used the **Set-DeviceNoteFriendlyName_GraphAPI.ps1** script, you can see which devices got updated.

![Output](https://cdrt.github.io/mk_blog/img/2023/intune_device_notes/image2.png)
