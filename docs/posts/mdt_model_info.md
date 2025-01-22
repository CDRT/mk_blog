---
date:
    created: 2020-03-30
authors:
    - Thad
categories:
    - "2020"
title: "Lenovo Model Information in MDT"
---

When using Microsoft Deployment Toolkit (MDT), the ZTIGather.wsf script will pull important information MDT needs and place it into variables for usage during a task sequence.   The %Model% variable is a key piece of information during task sequences especially for targeting software and drivers.<!-- more -->A popular example of the %Model% variable usage is the [Total Control](https://deploymentresearch.com/mdt-2013-lite-touch-driver-management/) method of driver organization and injection by Johan Arwidmark.


## Problem

Currently, when the model information is pulled from a Lenovo system, it will display the Lenovo Machine Type Model (MTM) in the Model variable as 20MD001YUS or 20MD003YUS, for example.  When using the MTM, the Out-of-Box Drivers library of model folders in MDT can quickly grow and become nearly repetitive.  The growth can become a management burden when attempting to maintain drivers, as each folder beginning with 20MD, for example, would have the same set of drivers in it.

## Resolution

To enable better management, we can edit the ZTIGather.wsf script to change where it pulls the model information on Lenovo computers.  The script change will set the %Model% variable to the friendly name from WMI, such as ThinkPad P1.

!!! Warning "Backup Script"
    Please make a backup copy of ZTIGather.wsf prior to editing.

In the ZTIGather.wsf, we are going to add a bit of logic to detect if the Make is Lenovo and act accordingly to set the model information.

**ZTIGather.wsf**

```VBScript
' Get the make, model, and memory from the Win32_ComputerSystem class

Set objResults = objWMI.InstancesOf("Win32_ComputerSystem")
For each objInstance in objResults
    If not IsNull(objInstance.Manufacturer) then
        sMake = Trim(objInstance.Manufacturer)
    End if
    If sMake <> "LENOVO" Then
        If not IsNull(objInstance.Model) then
            sModel = Trim(objInstance.Model)
        End if
    End if
    If not IsNull(objInstance.TotalPhysicalMemory) then
        sMemory = Trim(Int(objInstance.TotalPhysicalMemory / 1024 / 1024))
    End if
Next
If sMake = "" then
    oLogging.CreateEntry "Unable to determine make via WMI.", LogTypeInfo
End if
If sMake <> "LENOVO" Then
    If sModel = "" then
        oLogging.CreateEntry "Unable to determine model via WMI.", LogTypeInfo
    End if
End If

' Get the UUID from the Win32_ComputerSystemProduct class
Set objResults = objWMI.InstancesOf("Win32_ComputerSystemProduct")
For each objInstance in objResults
    If not IsNull(objInstance.UUID) then
        sUUID = Trim(objInstance.UUID)
    End if
    If sMake = "LENOVO" Then
        If not IsNull(objInstance.Version) then
            sModel = Trim(objInstance.Version)
        End if
    End if
Next
If sUUID = "" then
    oLogging.CreateEntry "Unable to determine UUID via WMI.", LogTypeInfo
End if

If sMake = "LENOVO" Then
    If sModel = "" then
        oLogging.CreateEntry "Unable to determine model via WMI.", LogTypeInfo
    End if
End if
```

While on the topic of model information and injecting drivers, there is an additional change to ensure that only the drivers for the model being deployed are applied.  While testing driver packs, we have found an instance where the incorrect driver folder may be used.  This is caused by one model name being an exact substring of a second model.  An example would be ThinkPad P1 and ThinkPad P1 Gen 2.  They both contain ThinkPad P1.

For this to be an issue, Out-of-Box Drivers repository would need to be alphabetized, ThinkPad P1 would need to be above ThinkPad P1 Gen 2 in the list of folders. When the DriverGroup001 variable is set to %Make%\%Model%, the ZTIConfigFile.vbs searches for the matching Out-of-Box Drivers folder structure to apply to the variable.  By default, the script applies the last found instance.  This means, in the above example, when deploying a Lenovo ThinkPad P1, the script will find Lenovo\ThinkPad P1 Gen 2 and assign it to the DriverGroup001 variable.  What needs to happen is for there to be an exact match.

In the ZTIConfigFile.vbs, there is one line of code to edit.  Editing this line of code will allow the script to precisely define the Out-Of-Box Drivers subfolder(s) when reading the DriverGroup001 variable.

!!! Warning "Backup Script"
    Please make a backup copy of ZTIConfigFile.vbs prior to editing.

**ZTIConfigFile.vbs**

Change the following line of vbscript code from
```VBScript
If InStr(1,sName & "\", oGroupItem, vbTextCompare ) <> 0 then
```
to
```VBScript
If InStr(1,sName & "\", oGroupItem, vbTextCompare ) <> 0 AND (LEN(sName) = LEN(oGroupItem)) then
```

The original line of code performs a compare to find if the DriverGroup001 variable matches any subfolders.  The issue in this line of code is that if the deployed model is a ThinkPad P1, but we have subfolders for the ThinkPad P1 and ThinkPad P1 Gen 2, the search will find both but use the last match.  If the DriverGroups.xml is organized alphabetically, then the last subfolder found in this example is the ThinkPad P1 Gen 2.  Since the example deployment was for a ThinkPad P1, this will result in the incorrect set of drivers being applied to the device.

The edited line of code retains the original search for the text from DriverGroup001, but now checks that the length of the DriverGroup001 variable is the same as the Out-Of-Box Drivers subgroup name being queried.  The result of the new line of code is that only the group being searched for will be used.