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
The below code can be used to accomplish this. If there's currently a note set for a device, the friendly name will be appended on the next line of the note. You can also download them from my GitHub: [Set-DeviceNoteFriendlyName_GraphSDK.ps1](https://github.com/philjorgensen/Graph/blob/main/Set-DeviceNoteFriendlyName_GraphSDK.ps1) and [Set-DeviceNoteFriendlyName_GraphAPI.ps1](https://github.com/philjorgensen/Graph/blob/main/Set-DeviceNoteFriendlyName_GraphAPI.ps1).

This was tested in a small tenant so results may vary as far as time to completion goes.

!!! info
    PowerShell 7 is recommended

## Graph PowerShell SDK cmdlets

This option uses the cmdlets from the [Graph PowerShell SDK](https://learn.microsoft.com/powershell/microsoftgraph/overview?view=graph-powershell-1.0).

```powershell
--8<-- "https://raw.githubusercontent.com/philjorgensen/Graph/main/Set-DeviceNoteFriendlyName_GraphSDK.ps1"
```

## Invoke-MgGraphRequest

This other option makes direct calls to the Graph API using [Invoke-MgGraphRequest](https://learn.microsoft.com/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest?view=graph-powershell-1.0).

```powershell
--8<-- "https://raw.githubusercontent.com/philjorgensen/Graph/main/Set-DeviceNoteFriendlyName_GraphAPI.ps1"
```

## Results

Once finished, check a device's notes in the portal to find it's friendly name.

![DeviceNote](https://cdrt.github.io/mk_blog/img/2023/intune_device_notes/image1.png)

If you used the **Set-DeviceNoteFriendlyName_GraphAPI.ps1** script, you can see which devices got updated.

![Output](https://cdrt.github.io/mk_blog/img/2023/intune_device_notes/image2.png)
