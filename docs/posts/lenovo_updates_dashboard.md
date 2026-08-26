---
date:
    created: 2026-08-25
authors:
    - Joe
categories:
    - "2026"
title: Introducing The Lenovo Updates Dashboard
---

Answering "what is the newest BIOS for this ThinkPad?" or "which of my models got that Intel graphics driver?" usually means opening the support site for one machine type at a time. The Lenovo Updates Dashboard reads the same public System Update catalogs that Lenovo's own update clients use, pools the results for every model you track, and gives you one searchable grid with CSV export.

<!-- more -->

![Lenovo Updates Dashboard Updates tab](https://cdrt.github.io/mk_blog/img/2026/lenovo_updates_dashboard/updates_tab.png)

## What it does

The dashboard is a self-contained PowerShell and WPF desktop application. You give it the 4-character machine types in your fleet, it downloads each model's Windows 11 catalog and every package descriptor that catalog references, and it stores the parsed results in a local JSON file.

From there you can browse, search, filter across models, and export. Each row is a package version, and the grid carries the details that actually drive a deployment decision:

| Column | Meaning |
|---|---|
| **Title** | Package title. Click it to open Lenovo's readme in your browser. |
| **Version** | Package version. Click it to open the raw descriptor XML. |
| **Category** | Lenovo's own category, such as *Display and Video Graphics*. |
| **Type** | Driver, BIOS, Firmware, Application, or Other. |
| **Reboot** | None, recommended reboot, forced reboot, forced shutdown, or delayed required reboot. |
| **Released** | The release date Lenovo published. |
| **Models** | Which of your tracked machine types are offered the package. |
| **Offered** | Ticked while the package is still in at least one selected model's catalog. |
| **Superseded** | Ticked when a newer version of the same package exists. |

Rows for packages Lenovo no longer offers are greyed out, and superseded versions are shown in italics. **Latest only** hides superseded versions; **Include no longer offered** brings withdrawn packages back into view so you can see what a model used to be offered. **Export CSV** writes exactly the rows currently displayed.

The **Categories** tab is a cross-model rollup of how many records you have collected all-time against how many are currently offered per category, which is the fastest way to spot where the churn in your fleet is. Double-click a category to jump to the Updates tab filtered to it.

!!! note
    The dashboard reads catalog metadata only. It never downloads an update payload, never installs anything, and never contacts or modifies the machines in your fleet.

## Requirements

| Requirement | Detail |
|---|---|
| Operating system | Windows 10 or Windows 11 |
| PowerShell | Windows PowerShell 5.1 (built into Windows) |
| .NET | .NET Framework 4.6 or later with WPF (built into Windows) |
| Network | HTTPS to `download.lenovo.com` |
| Privileges | Standard user. Administrator rights are not required. |

That single host is the only external dependency. Connections use TLS 1.2 and follow the proxy configured for the user account. There is no installer, no registry footprint, no agent, and no extra PowerShell module to deploy.

Catalog coverage is **Windows 11** for Lenovo commercial products: ThinkPad, ThinkCentre, and ThinkStation.

## Getting started

Download the package from [download link](#) and extract it anywhere you can write to, for example `C:\Tools\LenovoUpdatesDashboard`. Then launch it:

``` powershell
powershell.exe -STA -ExecutionPolicy Bypass -File ".\LenovoUpdatesDashboard.ps1"
```

`-STA` matters. WPF requires single-threaded apartment mode, and while the script will relaunch itself in STA when it can, supplying the switch avoids starting a second process.

If your execution policy is enforced centrally and the launch is still blocked, unblock the extracted files once:

``` powershell
Get-ChildItem -Path . -Recurse | Unblock-File
```

The dashboard opens on the **Models** tab with an empty list. Click **+ Add Model**, enter a machine type and a friendly name, repeat for each model you want to track, then click **Sync Now**.

### Finding a machine type

The machine type is the first four characters of the MTM on the product label, for example `21XF` in `21XF0025US`. On a running machine:

``` powershell
(Get-CimInstance Win32_ComputerSystemProduct).Name.Substring(0,4)
```

One machine type covers every configuration in that model line, so you track a handful of machine types rather than every individual MTM.

## What a sync actually does

1. Each tracked model's Windows 11 catalog is downloaded from `download.lenovo.com`.
2. The unique set of descriptor URLs is collected across all models. A descriptor is downloaded only if it is new or its catalog checksum changed, so a package shared by several models is fetched once and an unchanged package is never fetched again.
3. Descriptors are downloaded in parallel and parsed for package ID, name, version, title, release date, package type, and reboot type.
4. Records and model associations are added or refreshed. A package that has dropped out of a model's catalog is flagged *not offered* for that model, but the record is kept so you retain the history.
5. Supersession is recalculated across everything.

The first sync for a model is the slow one because every descriptor is new. Later syncs fetch only what changed. On a fast link, raise **Max concurrent downloads** on the Settings tab, which defaults to 12.

## Keeping the data current

`Sync-LenovoUpdates.ps1` performs the same sync with no window at all, against the same data the dashboard reads. Point a scheduled task at it and the dashboard is current whenever you open it.

``` powershell title="Register the daily sync task"
$script = "C:\Tools\LenovoUpdatesDashboard\Sync-LenovoUpdates.ps1"

$action  = New-ScheduledTaskAction -Execute 'powershell.exe' `
    -Argument "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$script`""
$trigger = New-ScheduledTaskTrigger -Daily -At (Get-Date -Hour 10 -Minute 0 -Second 0)
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable

Register-ScheduledTask -TaskName 'Lenovo Updates Sync' `
    -Action $action -Trigger $trigger -Settings $settings `
    -Description 'Refresh Lenovo update catalog data' `
    -User "$env:USERDOMAIN\$env:USERNAME" -RunLevel Limited -Force
```

Registering against the interactive user means no stored password, but the task then runs only while that user is logged on. Re-register the principal with a service account to run it regardless.

The script writes a summary to `data\sync.log` and sets an exit code the scheduler surfaces as **Last Run Result**:

| Exit code | Meaning |
|---|---|
| 0 | Success, no errors. |
| 1 | Completed, but one or more catalogs or descriptors had problems. |
| 2 | Nothing to do. No models are tracked. |
| 3 | Fatal error, for example the engine could not be loaded. |

It also takes `-MachineType 21XF,21VA` to sync a subset, and `-Force` to re-parse everything and ignore checksums.

### Toast notifications

Add `-Notify` to the scheduled task arguments and Windows raises a toast when a sync finds something new, reporting how many updates are new, how many of your models they affect, and a line per update in the form `Title Version - 21XF, 21VA`.

``` text
-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\Tools\LenovoUpdatesDashboard\Sync-LenovoUpdates.ps1" -Notify
```

`-NotifyAlways` also toasts when nothing new was found, which is useful for confirming the schedule is firing, and `-ToastMaxItems <n>` changes how many updates are listed before the rest collapse.

!!! warning
    Toasts need an interactive desktop session, so the task must be set to run only when the user is logged on. The first run registers a per-user identity under `HKCU\SOFTWARE\Classes\AppUserModelId` so the toast is branded correctly - no administrator rights are needed and nothing outside your own profile is touched. Focus assist and Do Not Disturb suppress toasts.

## Driving the engine from the console

Everything the GUI does is exposed as PowerShell commands, so the same data can feed your own reporting.

``` powershell
Import-Module ".\LenovoUpdatesDashboard.psm1" -Force

Add-TrackedModel -MachineType 21XF -Name 'ThinkPad X1 Carbon Gen 12'
Invoke-CatalogSync

Get-Update -SearchTitle 'Intel'
Get-Update -Category 'Display and Video Graphics'
Get-Update -Model 21XF -LatestOnly | Export-UpdateReport -Path .\report.csv
Get-CategorySummary
Get-DashboardStatus
```

`Get-Update` filters on `-Model`, `-Category`, `-SearchTitle`, `-PackageType`, `-IncludeUnoffered`, and `-LatestOnly`. `ConvertFrom-LnvCatalogXml` and `ConvertFrom-LnvDescriptorXml` will parse a catalog or descriptor from `-Xml` or `-Path` if you only want the parser. Every command supports `Get-Help`.

## Where the data lives

Everything sits in the `data` folder beside the scripts as plain JSON and plain text, so it is easy to back up, inspect, or move.

| File | Contents |
|---|---|
| `data\database.json` | Models, updates, and associations. |
| `data\database.json.bak` | Previous copy, rotated on every save. |
| `data\config.json` | Catalog URL, download concurrency, theme. |
| `data\sync.log` | Timestamped log of every sync and notable event. |

To move to another machine, copy the folder. To start over, delete `database.json` and run a sync. Because the data folder sits beside the scripts, the whole solution can live on a network share and be shared by several administrators against one data set. Only one of them should sync at a time.

## Summary

- No installer, no agent, no module dependencies. Extract and run.
- Standard user rights, one outbound HTTPS host, metadata only.
- Fleet-wide view of drivers, BIOS, firmware, and applications with supersession and reboot type surfaced per package.
- CSV export of whatever the grid is currently showing.
- Scheduled headless sync with optional toast notifications for new updates.
- Full PowerShell command surface for your own automation.
