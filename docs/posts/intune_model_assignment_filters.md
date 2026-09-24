---
date:
    created: 2026-09-24
authors:
    - Phil
categories:
    - "2026"
title: Creating Intune Assignment Filters for Lenovo Models
---

Intune reports `device.model` for Lenovo hardware as the machine type model — `21N1CTO1WW` — not the friendly name. Writing a model-based assignment filter means writing it against machine type prefixes, which first means working out every prefix that belongs to the model you care about. This post covers a PowerShell script that pulls those prefixes from the Lenovo model catalog and builds the filters for you.
<!-- more -->

## The problem

A few years back I covered [adding the friendly name to Intune device notes](https://blog.lenovocdrt.com/adding-model-friendly-name-to-intune-device-notes), which solves the reporting side. Targeting is the other half. An [assignment filter](https://learn.microsoft.com/intune/fundamentals/filters/overview) scopes an assignment by rules you create, and model targeting is a rule built on the `device.model` property.

A filter rule that targets a ThinkPad T14s Gen 6 has to look like this:

```
(device.model -startsWith "21M1") or (device.model -startsWith "21M2") or (device.model -startsWith "21N1") or ...
```

To write that by hand you need every machine type that ships under that name. There are ten, spread across three processor architectures, and nothing in the Intune portal will tell you what they are.

## The Lenovo model catalog

The [model catalog](https://download.lenovo.com/cdrt/td/catalogv2.xml) already has this mapping. Each entry pairs a name with its machine types:

``` xml
<Model name="ThinkPad T14S Gen 6 Type 21N1 21N2" arch="Qualcomm">
  <Types>
   <Type>21N1</Type>
   <Type>21N2</Type>
  </Types>
```

The catch is the `Type XXXX YYYY` suffix on the name. One friendly name is split across several entries, so T14s Gen 6 is five of them:

| Catalog entry | Architecture | Machine types |
|---|---|---|
| ThinkPad T14S Gen 6 Type 21M1 21M2 | AMD | 21M1, 21M2 |
| ThinkPad T14S Gen 6 Type 21N1 21N2 | Qualcomm | 21N1, 21N2 |
| ThinkPad T14S Gen 6 Type 21QX 21QY | Intel | 21QX, 21QY |
| ThinkPad T14S Gen 6 Type 21R1 21R2 | Intel | 21R1, 21R2 |
| ThinkPad T14S Gen 6 Type 21TB 21TC | AMD | 21TB, 21TC |

Strip the suffix and merge the machine types and you get one model covering all ten. Across the whole catalog that collapses 508 entries down to 398 models.

Rule length is not a concern. The largest model in the catalog is the ThinkCentre M70T Gen 3 at 16 machine types, which produces a rule of about 640 characters against Intune's 3072 character limit.

## Requirements

- The `Microsoft.Graph.Authentication` module
- The `DeviceManagementConfiguration.ReadWrite.All` Graph API permission scope
- [New-LnvAssignmentFilter.ps1](https://github.com/philjorgensen/Intune/blob/main/AssignmentFilters/New-LnvAssignmentFilter.ps1)

## Choosing what to target

Every run starts by picking models out of the catalog, either by name or by machine type. Add `-ListAvailable` to any of these and the script reads the catalog and prints what it found without touching Graph — worth doing before the first real run.

### By model name

The catalog has its own spelling of model names, so confirm one before building a pattern around it:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'T14* Gen 6' -ListAvailable
```

```
Model               Architecture         MachineTypeCount MachineType
-----               ------------         ---------------- -----------
ThinkPad T14S Gen 6 AMD, Intel, Qualcomm               10 21M1, 21M2, 21N1, 21N2, 21QX, 21QY, 21R1, 21R2, 21TB, 21TC
ThinkPad T14 Gen 6  AMD, Intel                          6 21QC, 21QD, 21QG, 21QH, 21QJ, 21QK
```

Drop the `Gen 6` and the pattern widens to the whole line — `-Model 'T14*'` returns all 16 T14 and T14s models in the catalog.

!!! note
    Catalog naming is not consistent, even inside a single product line. ThinkPad T14 uses `Gen 6` throughout, while X1 Carbon alone spans `8th Gen`, `10TH Gen` and `Gen 14`. Word forms drift from marketing too — the catalog writes `T14S 2IN1`, not `T14s 2-in-1`. Capitalization is the one thing you do not have to match, since `-Model` is case insensitive.

### Without the brand word

Typing `ThinkPad` in front of everything gets old. These all resolve to the same model:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'ThinkPad T14 Gen 6' -ListAvailable
.\New-LnvAssignmentFilter.ps1 -Model 'T14 Gen 6' -ListAvailable
.\New-LnvAssignmentFilter.ps1 -Model 't14 gen 6' -ListAvailable
```

Matching stays exact on the rest of the name, so `T14 Gen 6` does not drag in `T14S Gen 6`. Drop `-ListAvailable` and the filter is named from the full catalog name regardless of how it was matched, so you still get **Lenovo - ThinkPad T14 Gen 6**.

Each pattern is evaluated separately. One that matches nothing raises a warning and the run continues with the rest.

### By machine type

The reverse lookup is the one I reach for most. You are looking at a device in the portal, you have `21N1CTO1WW`, and you have no idea what it is. Paste it in as-is:

``` powershell
.\New-LnvAssignmentFilter.ps1 -MachineType '21N1CTO1WW' -ListAvailable
```

Only the first four characters matter, so the bare `21N1` works too. Wildcards are expanded against the catalog:

``` powershell
.\New-LnvAssignmentFilter.ps1 -MachineType '21L*' -ListAvailable
```

!!! warning
    A machine type selects the whole model it belongs to, so looking up `21N1` produces the complete T14s Gen 6 filter covering all ten machine types, not just the one you supplied. Machine type ranges also span product lines — `21L*` pulls in X13 and X12 Detachable alongside the L series.

## Creating the filter

`-WhatIf` shows what would be created without posting anything:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'ThinkPad T14S Gen 6' -WhatIf
```

Drop it to make the post call:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'ThinkPad T14S Gen 6'
```

That produces a single filter named **Lenovo - ThinkPad T14S Gen 6** on the `windows10AndLater` platform, with a rule covering all ten machine types.

Wildcards and multiple patterns work, so a whole product line is one command:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'ThinkPad X1 Carbon*', 'ThinkPad T14*'
```

### One filter per architecture

T14s Gen 6 ships on Intel, AMD and Qualcomm. When driver or firmware targeting has to keep those apart, `-SplitByArchitecture` creates one filter per architecture instead of one per model:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'ThinkPad T14S Gen 6' -SplitByArchitecture
```

Running the above command would generate this result:

```
Model       : ThinkPad T14S Gen 6
DisplayName : Lenovo - ThinkPad T14S Gen 6 (Qualcomm)
MachineType : 21N1, 21N2
Action      : Created
Detail      : 2 machine type(s) matched.

Model       : ThinkPad T14S Gen 6
DisplayName : Lenovo - ThinkPad T14S Gen 6 (AMD)
MachineType : 21M1, 21M2, 21TB, 21TC
Action      : Created
Detail      : 4 machine type(s) matched.

Model       : ThinkPad T14S Gen 6
DisplayName : Lenovo - ThinkPad T14S Gen 6 (Intel)
MachineType : 21QX, 21QY, 21R1, 21R2
Action      : Created
Detail      : 4 machine type(s) matched.
```

Because a machine type belongs to exactly one architecture, combining this with `-MachineType` narrows to that architecture without you needing to know which one it is:

``` powershell
.\New-LnvAssignmentFilter.ps1 -MachineType '21N2' -SplitByArchitecture -ListAvailable
```

`21N2` is a Qualcomm machine type, so only the Qualcomm third of the T14s Gen 6 comes back. The script says so rather than leaving you to work it out from the machine type count:

```
WARNING: 21N2 resolved to ThinkPad T14S Gen 6 (Qualcomm). The AMD variant (21M1, 21M2, 21TB, 21TC) is a separate filter and was not included.
WARNING: 21N2 resolved to ThinkPad T14S Gen 6 (Qualcomm). The Intel variant (21QX, 21QY, 21R1, 21R2) is a separate filter and was not included.

Model               Architecture MachineTypeCount MachineType
-----               ------------ ---------------- -----------
ThinkPad T14S Gen 6 Qualcomm                    2 21N1, 21N2
```

One warning per architecture left out. Add a machine type from each and all three filters come back with nothing to warn about:

``` powershell
.\New-LnvAssignmentFilter.ps1 -MachineType '21N2', '21M1', '21QX' -SplitByArchitecture -ListAvailable
```

## Keeping filters current

Lenovo adds machine types to existing models over time. Filters that already exist are left alone by default and reported as `Skipped`. Pass `-Force` to refresh their rules from the catalog:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'T14S Gen 6' -Force
```

Anything that already matches the catalog comes back as `Unchanged`, so this is safe to run on a schedule.

!!! warning
    `-Force` only governs filters that already exist. Models matched by the pattern that have no filter yet are created either way, so keep the pattern as tight as the filters you actually maintain — `-Model 'ThinkPad*' -Force` would create filters for all 266 ThinkPad models in the catalog.

Every run emits one object per model with an `Action` of `Created`, `Updated`, `Unchanged`, `Skipped`, or `Failed`, so a scheduled run can keep a record of what changed:

``` powershell
.\New-LnvAssignmentFilter.ps1 -Model 'T14* Gen 6' | Export-Csv -Path .\filters.csv -NoTypeInformation
```

## Summary

- Run `-ListAvailable` first to confirm the catalog name
- Use `-WhatIf` before the first real run
- `-MachineType` covers the case where you have an MTM and nothing else
- Add `-SplitByArchitecture` when Intel, AMD and Qualcomm need separate targeting
- Re-run with `-Force` periodically to pick up machine types Lenovo has added

Filters are created for the `windows10AndLater` platform with an `assignmentFilterManagementType` of `devices`. Display names come from the catalog verbatim, which spells it `T14S` where marketing uses `T14s` — rename in the portal if you desire.
