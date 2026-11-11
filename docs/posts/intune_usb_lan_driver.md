---
date:
    created: 2026-11-11
authors:
    - Phil
categories:
    - "2026"
title: Deploying the Lenovo USB3.0 LAN Driver for Docks With Intune
---

Lenovo's latest Realtek USB Ethernet driver package for docks, adapters, and monitors won't show up in Commercial Vantage. To maintain accuracy across various model dependencies, the Accessories team advised against publishing this update to the catalog Commercial Vantage reads, so getting it onto devices is up to you. The package also ships under a new product name and a new installer, and it does not remove the version it replaces. This post covers packaging it as an Intune Win32 app, writing registry detection rules for both versions, and using supersedence to remove the older ThinkPad USB Ethernet Adapter Driver, along with the log entries that point to each problem along the way.
<!-- more -->

## Two products, one driver

The older release installs as **ThinkPad USB Ethernet Adapter Driver** and is built with InstallShield. The new release installs as **Lenovo USB3.0 LAN Driver for Docks Adapters And Monitors** and is built with Inno Setup. Each registers its own Uninstall key and installs to its own folder:

| | Previous version | New version |
|---|---|---|
| Package | `u3etn15w` | `u3etn16w_v3` |
| Installer | InstallShield (InstallScript) | Inno Setup |
| DisplayVersion | 11.19.0603.2025 | 11.21.1009.2025 |
| Uninstall key | `HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{04201224-2B34-4EE7-862B-B7BBF89DB3AB}` | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{C8A7314B-B4E4-45F5-AF10-3ACDF7F8B5B1}_is1` |
| Install folder | `C:\Program Files (x86)\Lenovo\Thinkpad USB Ethernet Adapter Driver` | `C:\Program Files (x86)\Lenovo\Lenovo USB3.0 LAN Driver for Docks Adapters And Monitors` |

Installing the new version leaves the old product registered, so a device that upgrades ends up with both entries in **Installed apps** and both sets of driver packages in the driver store. Supersedence in Intune takes care of the cleanup.

!!! note
    This post assumes devices already have the previous package, `u3etn15w`, installed. The full version history is in the **Version Information** section of the package README.

The new key lives in the 64-bit registry view even though the files land in `Program Files (x86)`. The old key lives under `WOW6432Node`. That difference drives the detection rule settings below.

### Which driver gets installed

The package stages a different Realtek INF depending on the chip in the connected adapter:

| INF | Driver version | Chip |
|---|---|---|
| `rtu56cx22x64.inf` | 1156.21.1009.2025 | RTL8156 USB 2.5GbE (also RTL8153 GbE) |
| `rtu59cx22x64.inf` | 1159.21.1009.2025 | RTL815A / RTL8D5A USB 10GbE |

Both driver versions share the `21.1009.2025` suffix with the package's `DisplayVersion` of `11.21.1009.2025`, so a single registry rule on the Uninstall key covers the install whichever chip it targeted.

!!! note
    The installer writes `InstallLog.txt` to the install folder, ending in `Exit code: = 0`. That line reports the result of the last `pnputil` call only, and the file is overwritten on every run, so it is not a reliable detection source.

## Package the new version

Download the package from the [Lenovo USB3.0 LAN Driver for Windows 11 download page](https://support.lenovo.com/downloads/ds504778). At the time of writing, `u3etn16w_v3.exe` is the latest version.

!!! tip
    This driver can also be deployed with [Windows Autopatch driver and firmware updates](https://learn.microsoft.com/windows/deployment/windows-autopatch/manage/windows-autopatch-manage-driver-and-firmware-updates). Autopatch delivers the driver package itself, without the Inno Setup installer, so it doesn't create the Uninstall key used for detection here or remove the previous `u3etn15w` product.

Wrap the installer with `IntuneWinAppUtil.exe` and create the Win32 app with these commands:

| Setting | Value |
|---|---|
| Install command | `u3etn16w_v3.exe /verysilent` |
| Uninstall command | `u3etn16w_v3.exe /verysilent /uninstall` |
| Install behavior | System |

The install command must match the setup file name inside the `.intunewin` exactly.

The uninstall command is the one documented in the package README. IME downloads the app content again before running an uninstall, so the setup file is available in the IME cache when the command runs.

### Detection rule

| Setting | Value |
|---|---|
| Rule type | Registry |
| Key path | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{C8A7314B-B4E4-45F5-AF10-3ACDF7F8B5B1}_is1` |
| Value name | `DisplayVersion` |
| Detection method | Version comparison |
| Operator | Greater than or equal to |
| Value | `11.21.1009.2025` |
| Associated with a 32-bit app on 64-bit clients | No |

`DisplayVersion` is a `REG_SZ` string, and **Version comparison** parses it into four numeric parts before comparing, so `11.21.1009.2025` correctly evaluates higher than something like `11.9.0.0`.

Use **Greater than or equal to** in place of **Equals**. If Windows Update or a technician installs a later release, an **Equals** rule reports the app as missing and a Required assignment runs the 11.21 installer again.

## Configure the previous version for supersedence

Supersedence runs the uninstall command and detection rule defined on the **old** Intune app, so that app needs both configured correctly. It does not need an assignment of its own. The new app's assignment drives the whole process.

### Detection rule

| Setting | Value |
|---|---|
| Rule type | Registry |
| Key path | `HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{04201224-2B34-4EE7-862B-B7BBF89DB3AB}` |
| Value name | `DisplayVersion` |
| Detection method | Version comparison |
| Operator | Greater than or equal to |
| Value | `11.19.0603.2025` |
| Associated with a 32-bit app on 64-bit clients | No |

The explicit `WOW6432Node` path with the 32-bit toggle set to **No** reads the key where it physically lives, matches what Regedit shows, and puts the exact path IME checks into `AppWorkload.log`. That makes it the easier form to troubleshoot.

### Uninstall command

Use the uninstall command from the package README:

``` text
u3etn15w.exe /verysilent /uninstall
```

The command must match the setup file name inside the old app's `.intunewin`. An uninstall that stops at a dialog hangs under SYSTEM and fails the supersedence step, so test the command from a SYSTEM context before you deploy.

### Supersedence

On the new app's **Supersedence** tab, add the old app and set **Uninstall previous version** to **Yes**. When the new app targets a device, IME checks the old app's detection rule, runs its uninstall command if it's detected, confirms the key is gone, and then installs the new version.

## Assign to ThinkPad devices

Assign the new app to device groups with an **include** assignment filter that limits it to ThinkPad models. Intune reports `device.model` as the machine type model (for example `21N1CTO1WW`), so a ThinkPad filter rule matches on machine type prefixes. [Creating Intune Assignment Filters for Lenovo Models](intune_model_assignment_filters.md) covers a script that builds these filters from the Lenovo model catalog.

## Troubleshooting

All of these entries come from `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\AppWorkload.log`.

### Install fails with error 2

``` text
[Win32App] lastWin32Error 2 after CreateProcess
[Win32App] lastHResult -2147024894 after CreateProcess
[Win32App] Failed to create installer process. Error code = 2
```

Error 2 (`0x80070002`) is file not found. IME could not find the file named in the install command inside `C:\WINDOWS\IMECache\<app id>_<version>`. Match the command to the file name in the package, including any subfolder, or rename the file and rebuild the `.intunewin`.

### The old version stays installed

Search the log for the old app's policy ID and its product code:

``` powershell
Select-String -Path 'C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\AppWorkload*.log' -Pattern '<old app id>|04201224' |
    Select-Object -Last 40 Line
```

| Log entry | Meaning |
|---|---|
| `Checked reg path: ...{04201224-...}` with `applicationDetected: True` | IME found the old version. |
| `applicationDetected: False` | IME didn't find it and skipped the uninstall. Check the old app's detection rule. |
| `Handler invoked with execution type: Uninstall` for the old app | IME ran the old app's uninstall command. The following lines show the command line and exit code. |
| `Not sending status update ... because the app does not have available, required, or uninstall intent` | Expected for the superseded app, since it has no assignment of its own. It confirms the supersedence relationship reached the device. |

If the old app reports `applicationDetected: False` while the key is present, confirm the key with:

``` powershell
reg query "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{04201224-2B34-4EE7-862B-B7BBF89DB3AB}" /v DisplayVersion
```

Then set the old app's detection rule to the explicit `WOW6432Node` path with the 32-bit toggle set to **No**, as shown above. After saving the change, restart the **Microsoft Intune Management Extension** service on the test device so it picks up the updated rule before the next detection cycle.

## Summary

- Detect the new version on `{C8A7314B-B4E4-45F5-AF10-3ACDF7F8B5B1}_is1` in the **64-bit** view, `DisplayVersion` **≥ 11.21.1009.2025**
- Detect the old version on the explicit **`WOW6432Node`** path for `{04201224-2B34-4EE7-862B-B7BBF89DB3AB}`, 32-bit toggle **No**
- Uninstall the new version with the setup file's **`/verysilent /uninstall`** switches from the README
- Uninstall the old version with **`u3etn15w.exe /verysilent /uninstall`** from the README
- Set **Uninstall previous version** to **Yes** on the new app's supersedence, and leave the old app **unassigned**
- Assign the new app with an **include filter for ThinkPad models**
- Check the Uninstall key on every future release before reusing these rules, since this release changed the product name, installer, and GUID
