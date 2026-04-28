---
date:
    created: 2026-04-28
authors:
    - Joe
categories:
    - "2026"
title: "Community Tool Spotlight: PowerBios – WMI BIOS Utility for Lenovo"
---

**PowerBios** is a free, community-developed Windows utility by Claude Boucher (SomeTools) that lets you read, modify, and export BIOS settings on Lenovo ThinkPad, ThinkCentre, and ThinkStation devices using the WMI interface — locally or remotely. It's not an official Lenovo product, but it's a genuinely useful tool for anyone who regularly works with Lenovo BIOS configuration.

<!-- more -->

!!! note
    PowerBios is a third-party community tool, not an official Lenovo product. Use it in accordance with your organization's software policies.

![PowerBios Config section showing BIOS settings, Supervisor Password, and Script Generation panels](https://cdrt.github.io/mk_blog/img/2026/powerbios_community_spotlight/main.png)

## What It Does

PowerBios connects to a target machine via WMI and exposes the full BIOS setting tree in a structured GUI — the same settings accessible through `Lenovo_BiosSetting` and `Lenovo_SetBiosSetting` WMI classes. From there you can:

- Browse and change individual BIOS settings with toggle switches and dropdowns
- Manage boot order with drag-and-drop reordering
- Handle Supervisor Password entry (both legacy `ascii,US` and the newer `Lenovo_WmiOpcodeInterface` method)
- Generate a ready-to-deploy `.ps1` script from the current selection
- Export a configuration record with machine identification (MTM, serial, BIOS version)

The script generator is the standout feature. After selecting the settings you want to apply, PowerBios writes a complete PowerShell script that handles apply, password submission, and save — with optional logging to console, individual per-machine log files, or a shared CSV for fleet-wide visibility.

## Getting It

PowerBios is available three ways:

| Method | Version | Architecture |
|---|---|---|
| Microsoft Store | 3.3.1.0 | x64 / ARM64 |
| Standalone `.exe` / `.zip` | 3.3.3.0 | x64, ARM64 |
| winget | 3.3.1.0 | x64 / ARM64 |

```powershell title="Install via winget"
winget install --name powerbios --source msstore
```

For standalone downloads, right-click the file after downloading, go to **Properties**, and check **Unblock** before running.

The [PowerBios download page](https://sometools.eu/PowerBios/) has direct links for all options.

## Connecting to a Device

Press **F2** (or the **F2 Previous Values** button) to initiate a WMI connection and load the device's BIOS data.

- **Local:** No credentials needed — a local admin account is sufficient.
- **Remote:** Provide the computer name or IP, plus a local admin username and password. Domain-joined machines on the same domain authenticate automatically. WMI Remote Access must be enabled on the target — see the [enable remote WMI guide](https://sometools.eu/enable-remote-wmi/) if needed.

Previously used remote machines are saved locally, so reconnecting is quick.

## Script Generator

The script generator is what makes PowerBios useful for actual deployment work rather than just ad-hoc configuration. Select the BIOS items you want to set, configure the script options, and PowerBios produces a `.ps1` that:

1. Applies each selected setting via WMI
2. Submits the Supervisor Password if required
3. Saves the BIOS settings

Script options worth knowing:

- **Check Model Type** — adds an MTM check at the top so the script only runs on the intended model
- **Display Log** — prints per-step results to the PowerShell console at runtime
- **Write Log (Detail)** — creates individual log files per machine, named `MTM_MachineName_YYYY-MM-DD_HH-MM.log`
- **Write Log (Global)** — appends results to a shared CSV file across all deployments, with a companion `_Monitor.ps1` that displays incoming results in real time

The global CSV logging approach is particularly useful for tracking BIOS configuration compliance across a fleet without any external tooling.

## Unknown BIOS Items

If PowerBios encounters a BIOS setting on the connected device that isn't in its database, it flags it with a warning indicator and shows it in a dedicated tab. Unknown items are fully functional — you can select, modify, and script them just like known items.

The tab also generates a report (machine ID + item list) that can be submitted to the PowerBios team at [support@sometools.eu](mailto:support@sometools.eu) for inclusion in a future update.
