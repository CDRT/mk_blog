---
date:
    created: 2026-06-08
authors:
    - Phil
categories:
    - "2026"
title: Current Drivers, Firmware, and BIOS During Autopilot Pre-Provisioning With Intune and LCU
slug: autopilot-pre-provisioning-current-drivers-firmware-bios-lcu
---

This post replaces the earlier [Autopilot + Thin Installer](https://blog.lenovocdrt.com/autopilot--thin-installer--current-driversbiosfirmware/) method. Thin Installer still works, but the [Lenovo.Client.Update](https://docs.lenovocdrt.com/guides/lcu/) (LCU) PowerShell module is the current, supported way to drive updates from a script: it queries, downloads, and silently installs applicable driver, firmware, and BIOS updates for a Lenovo device, without the dependency of Thin Installer. Wrapping it in a script and packaging that as an Intune Win32 app lets you run the same update pass unattended — including during an Autopilot for pre-provisioned [deployment](https://learn.microsoft.com/autopilot/pre-provision). This post walks through a wrapper script built for exactly that, and the packaging and detection rules that make it behave inside Intune.
<!-- more -->

## Why a Win32 app

LCU is published to the PowerShell Gallery, but calling `Install-Module` on a freshly imaged device during ESP is fragile — it depends on the NuGet package provider and PowerShellGet being present and able to bootstrap in SYSTEM context before the user reaches the desktop. Bundling the module inside the `.intunewin` sidesteps that: the module travels with the app, imports from a local path, and pins a known version.

Running it as a Win32 app (rather than a platform script) also gets you the things ESP needs — a real exit code, a `3010` reboot signal, and a detection rule the ESP can track during pre-provisioning.

## What the script does

The wrapper ([`Invoke-LenovoUpdate.ps1`](https://github.com/philjorgensen/Intune/blob/main/Autopilot/Lenovo-Client-Update/Invoke-LenovoUpdate.ps1)) handles the operational details around LCU:

- Confirms the device is a Lenovo system via `Win32_ComputerSystem` and exits cleanly on anything else.
- Imports the LCU module shipped alongside the script, falling back to an installed copy.
- Queries, downloads, and installs applicable **silent** updates in repeated passes until none remain.
- Verifies each package's Lenovo digital signature before installing (the LCU default).
- Writes a transcript log under `C:\ProgramData\Lenovo\Lenovo.Client.Update\Logs`.
- Returns an Intune-aware exit code and writes a registry detection marker.

### Why multiple passes

Installing one update can make a further update applicable — a chipset or base driver can unlock a dependent package that wasn't offered on the first query. The script loops, re-querying after each round, until a round finds nothing (or it hits `MaxRounds`, default 3).

``` powershell title="Invoke-LenovoUpdate.ps1"
    for ($round = 1; $round -le $MaxRounds; $round++)
    {
        Write-LcuLog "Round $round of $MaxRounds : querying available updates..."

        $updates = @(Get-LnvUpdate | Where-Object {
                $_.IsApplicable -and
                -not $_.IsInstalled -and
                $_.Installer.Unattended -and
                ($IncludeBIOS -or -not (Test-LnvBiosUpdate -Update $_)) -and
                -not $processed.Contains(('{0}|{1}' -f $_.ID, $_.Version))
            })

        Write-LcuLog "$($updates.Count) new applicable, unattended update(s) found this round."
        if ($updates.Count -eq 0) { break }

        # Download everything for this round first, then install.
        $updates | Save-LnvUpdate | Out-Null
    # ...install each, collecting results...
}
```

The filter is deliberately narrow: only packages that are applicable, not already installed, and flagged `Unattended`. Anything that would prompt or require interaction is skipped — there's no one at the keyboard during ESP.

### BIOS is opt-in

BIOS updates are excluded by default. Flashing the BIOS mid-Autopilot is risky: it needs AC power and a controlled reboot, neither of which you can guarantee on a device that just came out of the box. The `-IncludeBIOS` switch turns them on for targeted, AC-powered rollouts.

Identifying a BIOS-class package isn't a single property check, because the catalog represents it two ways:

| Model generation | `Type` | How it's matched |
|---|---|---|
| Older | `BIOS` | `Type` equals `BIOS` |
| Newer | `Firmware` | Title matches `BIOS`/`UEFI`/`System Firmware Update Utility`, or `RebootType` is `5` |

Scoping the `RebootType 5` check to `Type 'Firmware'` keeps it from catching ordinary firmware (docks, Thunderbolt) or regular drivers. When a BIOS-class package is installed, the script passes `-SaveBIOSUpdateInfoToRegistry` so the pending action lands under `HKLM\SOFTWARE\LenovoUpdate\BIOSUpdate` for follow-up detection.

![BIOS Registry Marker](https://cdrt.github.io/mk_blog/img/2026/intune_lcu_driver_updates/image1.png)

## Exit codes and reboots

After all rounds complete, the script tallies results and translates them to an exit code Intune understands:

- `0` — success, no reboot needed.
- `3010` — success, but at least one update reported `REBOOT_SUGGESTED`, `REBOOT_MANDATORY`, or `SHUTDOWN`. Intune (and the ESP) treats `3010` as a soft reboot and handles the restart.
- `1` — a fatal error was caught; check the transcript.

!!! note
    Configure the app's **Device restart behavior** as *Determine behavior based on return codes* so the `3010` is honored instead of being forced or suppressed.

## Detection

On a successful run the script writes a marker to `HKLM\SOFTWARE\LenovoUpdate\DriverUpdate`:

| Value | Meaning |
|---|---|
| `LastRun` | ISO 8601 timestamp of the run |
| `LastExitCode` | The exit code returned |
| `InstalledCount` | Packages installed this run |
| `FailedCount` | Packages that failed |
| `RebootRequired` | `1` if a reboot is pending |

For a one-shot Autopilot pass, a Registry detection rule of **value `LastRun` exists** is enough. To re-run on a schedule, use a custom detection script that only reports *detected* when `LastRun` falls inside your freshness window — anything older than that is treated as "not installed" and triggers another pass.

![Driver Registry Marker](https://cdrt.github.io/mk_blog/img/2026/intune_lcu_driver_updates/image2.jpg)

## Prepare the source directory

Organize the module and the wrapper script into a single source directory that will be packaged for deployment.

1. Create the source directory: `C:\Source\IntuneWin32\LCU`
2. Use the **Save-Module** cmdlet to download LCU into the source directory. This pulls the versioned module folder straight from the PowerShell Gallery — no manual file copying.

    ``` powershell
    Save-Module -Name Lenovo.Client.Update -Path "C:\Source\IntuneWin32\LCU"
    ```

3. Download [`Invoke-LenovoUpdate.ps1`](https://github.com/philjorgensen/Intune/blob/main/Autopilot/Lenovo-Client-Update/Invoke-LenovoUpdate.ps1) and place it in the root of the source directory.

The script imports whichever module version it finds next to itself, so `Save-Module` and the script don't need to agree on a version number ahead of time.

**Directory structure**

``` text
C:\Source\IntuneWin32\LCU\
    Lenovo.Client.Update\
        <ModuleVersion>\
            ├─ Lenovo.Client.Update.psd1
            ├─ Lenovo.Client.Update.psm1
            └─ ...
    Invoke-LenovoUpdate.ps1
```

!!! note
    To refresh the bundled module later, delete the `Lenovo.Client.Update` folder and re-run `Save-Module`, then rebuild the `.intunewin`.

## Packaging

1. Build the `.intunewin` from the source directory with the [Microsoft Win32 Content Prep Tool](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool):

    ``` text title="Package command"
    IntuneWinAppUtil.exe -c "C:\Source\IntuneWin32\LCU" -s Invoke-LenovoUpdate.ps1 -o "C:\Source\IntuneWin32\" -q
    ```

2. Use an install command that forces the 64-bit PowerShell host, so the module's native bits load correctly:

    ``` text title="Install command"
    %SystemRoot%\Sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -NoProfile -File .\Invoke-LenovoUpdate.ps1 -IncludeBIOS -ExportToWMI
    ```

!!! warning
    `Sysnative` is required. If a 32-bit PowerShell launches the script, the LCU module's native components fail to load.

### Create the Win32 app in Intune

With the `.intunewin` built, add it in the [Intune admin center](https://intune.microsoft.com):

1. Go to **Apps** → **Windows** → **Add**, and choose **Windows app (Win32)** as the app type.
2. On **App information**, upload the `.intunewin` package, then fill in **Name**, **Publisher**, and any other fields you want shown in Company Portal.
3. On **Program**, set the commands and behavior:
    - **Install command** — the `Sysnative` line from above.
    - **Uninstall command** — there's nothing to remove, so use a no-op such as `%SystemRoot%\Sysnative\cmd.exe /c reg delete "HKLM\SOFTWARE\LenovoUpdate" /f`.
    - **Install behavior** — **System**.
    - **Device restart behavior** — **Determine behavior based on return codes**, so the script's `3010` is honored.
4. On **Requirements**, set **Operating system architecture** to **64-bit** and a **Minimum operating system** version that matches your fleet. Also add the **OOBE-only requirement rule** below so the app is applicable only during pre-provisioning/OOBE.
5. On **Detection rules**, choose **Manually configure detection rules** and add a **Registry** rule: key `HKEY_LOCAL_MACHINE\SOFTWARE\LenovoUpdate\DriverUpdate`, value `LastRun`, detection method **Value exists**. (See [Detection](#detection) for the recurring-run alternative.)
6. Leave **Dependencies** and **Supersedence** empty, set **Assignments** on the next page (see [Deployment](#deployment)), then **Review + create**.

### Requirement rule: run only during OOBE/Autopilot

Because this is a pre-provisioning one-shot, add a script-based requirement rule so the app is only **applicable** while the device is still in OOBE. Once OOBE completes, the rule returns `No` and the app drops out of scope — another safeguard (alongside device-only assignment and a stable detection rule) against the multi-round driver sweep re-running during the user ESP.

- **Rule type:** Script
- **Run script as 32-bit process on 64-bit clients:** No
- **Return type:** Boolean
- **Operator:** Equals → **No**

??? example "Requirement Rule Script"

    ```powershell
    $Definition = @"

    using System;
    using System.Text;
    using System.Collections.Generic;
    using System.Runtime.InteropServices;

    namespace Api
    {
        public class Kernel32
        {
            [DllImport("kernel32.dll", CharSet = CharSet.Auto, SetLastError = true)]
            public static extern int OOBEComplete(ref int bIsOOBEComplete);
        }
    }
    "@

    Add-Type -TypeDefinition $Definition -Language CSharp

    $IsOOBEComplete = $false
    $appRequirement = [Api.Kernel32]::OOBEComplete([ref] $IsOOBEComplete)

    if ($IsOOBEComplete -eq '1') { return $true }
    else { return $false }
    ```

The rule returns `$true` once OOBE is complete; with the operator set to **Equals → No**, the app is applicable only while the device is still in OOBE. Credit to [Michael Niehaus](https://oofhours.com/2023/09/15/detecting-when-you-are-in-oobe/) for the `OOBEComplete` approach.

## Deployment

This is built for **Autopilot pre-provisioning** (the technician/white-glove phase), where the device is wired, on AC power, and nobody is waiting at the keyboard — so long downloads and multiple reboots are acceptable. That's also the only phase where enabling `-IncludeBIOS` makes sense.

### Assign to a device group

Assign the app to a **device** group, not a user group. User-targeted Win32 apps can't install during the technician phase — they run during account setup (the user-facing ESP), which would re-run the entire multi-round driver sweep with the user waiting at the screen.

### Don't let it block the user ESP

This is the part that trips people up. The ESP **blocking-apps list is shared across phases** — there is no way to block only the technician phase. Any app you add to *Block device use until these required apps are installed* is also tracked during account setup, so the user ESP waits on it again even though pre-provisioning already installed it.

Because the work is already done in pre-provisioning, keep it out of the user phase's path:

- In the ESP profile, use **Selected** blocking apps (not **All**), and **leave this app out** of that selection. It still installs as a required device app during the technician phase; it just no longer holds the user at the ESP.
- Keep the detection rule **stable and machine-scoped** so the account setup phase sees it as already installed and never reinstalls. The `HKLM\SOFTWARE\LenovoUpdate\DriverUpdate` marker with a **`LastRun` exists** registry rule is instant and reliable in both phases (the IME evaluates it as 64-bit SYSTEM). Prefer this over the freshness-window detection script for a pre-provisioning one-shot, which could report *not detected* and trigger a reinstall during user ESP.

!!! note
    If user ESP is taking a long time even though pre-provisioning completed successfully, it's almost always one of two things: the app is **user-assigned** (so it reinstalls during account setup), or it's still in the **blocking-apps list** (so the user phase waits on it). Device-only assignment plus a stable detection rule, and excluding it from the blocking selection, resolves both.

## Results

Before resealing the device, you can take a peek at the log to see which components have been updated.

![Log](https://cdrt.github.io/mk_blog/img/2026/intune_lcu_driver_updates/image3.jpg)

Additional session files can be found under `C:\ProgramData\Lenovo\Lenovo.Client.Update\History` that shows the update's package Id and version.

## Summary

A reliable unattended setup:

- Package the **LCU module folder with the script**
- Run via **`Sysnative` 64-bit PowerShell**.
- Leave **BIOS off** unless the rollout is targeted and AC-powered (`-IncludeBIOS`).
- Set restart behavior to **Determine behavior based on return codes** so `3010` works.
- Detect on the **`DriverUpdate` registry marker** — `LastRun` exists for one-shot, or a freshness check for recurring runs.
- **Assign to a device group** and target **pre-provisioning** — keep it out of the ESP blocking-apps selection so it doesn't hold up the user ESP.
