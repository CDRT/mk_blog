---
date:
    created: 2026-03-30
authors:
    - Joe
categories:
    - "2026"
title: Introducing the ThinkVantage PowerShell Library
---

<!--
![ThinkVantage](https://cdrt.github.io/mk_blog/img/2026/thinkvantage_powershell_library/thinkvantage_logo.png){ width="360" } -->

![ThinkVantage](../img/2026/thinkvantage_powershell_library/thinkvantage_logo.png){ width="360" }

The ThinkVantage PowerShell Library is a collection of PowerShell modules for managing Lenovo ThinkPad, ThinkCentre, and ThinkStation fleets at scale. It covers the full lifecycle — driver and firmware updates, BIOS configuration, and certificate-based BIOS authentication — all from the command line or scripted into your deployment workflows.
<!-- more -->

The library consists of these modules and scripts available in the PowerShell Gallery:

- **New**{ .badge-new } **Lenovo.Client.Update** module
- **Lenovo.BIOS.Config** module
- **Think BIOS Config Tool** script
- **Lenovo.Bios.Certificates** module
- **Lenovo BIOS Certs Tool** script
- **Lenovo.Client.Scripting** module

Each can be installed independently and used on its own, but together they provide a unified toolkit for automating Lenovo device management across Intune, Configuration Manager, and bare-metal OSD environments.


## The Four Modules

### Lenovo.Client.Update

The Lenovo fork of [LSUClient](https://www.powershellgallery.com/packages/LSUClient). Discover, download, and install driver, BIOS/UEFI, and firmware updates on Lenovo commercial PCs. Ships with 12 cmdlets and includes support for local update repositories, OSD task sequences, and digital signature verification.

``` powershell
Install-Module 'Lenovo.Client.Update'
```

### Lenovo.BIOS.Config

Read and modify BIOS settings via WMI. Export and import INI-based configuration profiles with optional encrypted passwords. Package BIOS configurations as Win32 apps or Remediations and deploy through Microsoft Intune — including direct upload via Microsoft Graph.

``` powershell
Install-Module -Name Lenovo.BIOS.Config
```

This module is a dependency for the **Think BIOS Config Tool v2** script that provides a graphical user interface for ease of use. This script can also be installed from the PowerShell Gallery.

``` powershell
Install-Script -Name ThinkBiosConfigUI
```

### Lenovo.Bios.Certificates

Replace Supervisor passwords with X.509 certificates. BIOS commands are cryptographically signed using local key files or Azure Key Vault — the private key never leaves its secure store. Supports PEM, DER, and PFX formats, with batch processing for INI config files.

``` powershell
Install-Module -Name Lenovo.BIOS.Certificates
```

This module is also a dependency for the Lenovo BIOS Certificates Tool which provides the graphical user interface for working with BIOS Certificates. It can be installed from the PowerSehl Gallery as well.

``` powershell
Install-Script -Name LnvBiosCertInterface
```

### Lenovo.Client.Scripting

Collect useful information and perform simplified actions that can be leveraged in day to day management of Lenovo commercial PCs (ThinkPad, ThinkCentre, ThinkStation). Use intuitive Get- cmdlets to retrieve information from devices directly, or use Find- cmdlets to get information from Internet-hosted resources.

``` powershell
Install-Script -Name Lenovo.Client.Scripting
```

!!! note
    All four modules require **Windows 10/11**, **PowerShell 5.0+**, and **Administrator** privileges.

## Why Automate?

- **Consistent, repeatable deployments** — Script-driven updates eliminate manual steps and configuration drift across your device fleet
- **Full PowerShell pipeline support** — Cmdlets are designed for chaining. Discover, download, and install in a single pipeline
- **Intune and Configuration Manager ready** — Package BIOS configurations as Win32 apps or Remediations and deploy via Intune with MS Graph upload
- **Enterprise security built in** — Digital signature verification, certificate-based BIOS authentication, and Azure Key Vault integration
- **OSD optimized** — Dynamically install applicable updates during bare-metal deployments without maintaining static driver packages

## Lenovo.Client.Update

Lenovo has published the Lenovo.Client.Update module as our own modified version of the popular LSUClient solution by jantari. The Lenovo version of this solution provides digitally signed PowerShell scripts as well as digital signature verification of the Lenovo update packages. Additional features have been added as well as some cmdlets moved into this module from the Lenovo.Client.Scripting module to consolidate the Lenovo updates related functions.

Read more about it in the [Product Guide](https://docs.lenovocdrt.com/guides/lcu/lcu_top.md)

### Core Cmdlets

<div class="first-col-nowrap" markdown>

| Cmdlet | Purpose |
|---|---|
| `Get-LnvUpdate` | Query applicable updates for the current device or a target model. Returns only needed updates by default |
| `Save-LnvUpdate` | Download packages to a local directory with optional progress display. Supports custom repositories |
| `Install-LnvUpdate` | Install packages silently. Use `-ExportToWMI` to log install history |
| `Get-LnvUpdatesRepo` | Build a local update repository filtered by package type and reboot behavior — ideal for OSD staging |
| `Get-LnvUpdateSummary` | Get an instant snapshot of update compliance status. Useful in Intune Remediations |
| `Get-LnvUpdateHistory` | Retrieve full installation history filtered by date range for compliance reporting |

</div>

### Update Workflow in 3 Lines

``` powershell title="update-workflow.ps1"
# Discover applicable updates for this machine
$updates = Get-LnvUpdate
$updates | Format-Table -Property Title, Category, ReleaseDate

# Download to a local staging directory
$updates | Save-LnvUpdate -Path "C:\Lenovo\Updates" -ShowProgress

# Install with verbose output
$updates | Install-LnvUpdate -Verbose
```

Filter for silent-only packages before installing with `Where-Object { $_.Installer.Unattended }`.

### OSD Task Sequence

For bare-metal deployments, stage a local repository and run two installation passes to catch chained driver dependencies:

``` powershell title="osd-task-sequence.ps1"
# Stage a local repo — exclude auto-restart packages (reboot type 1)
$repo = 'C:\Lenovo_Updates'
Get-LnvUpdatesRepo -RepositoryPath $repo -RebootType '0,3,5'

# First pass: install applicable updates, log history to WMI
Get-LnvUpdate -Repository $repo |
    Install-LnvUpdate -Path $repo -ExportToWMI -Verbose

# Second pass: catches chained driver dependencies
Get-LnvUpdate -Repository $repo |
    Install-LnvUpdate -Path $repo -ExportToWMI
```

!!! note
    Run `Get-LnvUpdate` twice — some drivers only become applicable after a prerequisite is installed.

### Security

- **Enforced installation signing** — By default `Install-LnvUpdate` will block unsigned or tampered packages


### Migrating from LSUClient

Lenovo.Client.Update is API-compatible with LSUClient. The migration is straightforward — rename your cmdlet calls and gain all the new security and repository features:

| LSUClient | Lenovo.Client.Update |
|---|---|
| `Get-LSUpdate` | `Get-LnvUpdate` |
| `Save-LSUpdate` | `Save-LnvUpdate` |
| `Install-LSUpdate` | `Install-LnvUpdate` |

## Think BIOS Config Tool V2

Think BIOS Config Tool V2 is a PowerShell WPF GUI and companion module (`Lenovo.BIOS.Config`) for reading, modifying, and deploying BIOS settings on Lenovo commercial PCs via WMI. It replaces the older HTA-based Think BIOS Config Tool.

### Features

| Feature | Description |
|---|---|
| **Settings Panel** | View and modify all BIOS settings interactively. Changed values are highlighted until saved |
| **Export Configuration** | Snapshot current BIOS settings to an INI file with optional encrypted Supervisor password |
| **Import Configuration** | Apply a saved INI profile to a target device. Handles SVP-protected settings automatically |
| **Intune Packaging** | Generate Win32 apps or Remediation packages from an INI. Optionally upload directly to Intune via Microsoft Graph |
| **Password Management** | Change, clear, or create password-change files for remote Supervisor Password management |
| **Custom Defaults** | Save settings as a Custom Defaults baseline — revert any device to your corporate BIOS standard |

### CLI Workflow

``` powershell title="bios-deploy-workflow.ps1"
# Export current BIOS settings to an INI file
Export-LnvWmiSettings -ConfigFile "C:\Temp\corp-bios.ini" -NoKey

# Apply INI to a target device (prompts for Supervisor Password)
$svp = Read-Host -AsSecureString 'Supervisor Password'
Import-LnvWmiSettings -ConfigFile 'C:\Temp\corp-bios.ini' -Current $svp

# Launch the GUI (elevated terminal required)
ThinkBIOSConfigUI   # Actions → Create Intune Package → Upload
```

Install the GUI script separately:

``` powershell
Install-Script 'ThinkBiosConfigUI'
```

## Lenovo.Bios.Certificates

For environments moving beyond Supervisor passwords, `Lenovo.Bios.Certificates` enables certificate-based BIOS authentication. BIOS commands are signed with an X.509 certificate — the private key can remain in Azure Key Vault or a local secure store, and no password is ever transmitted to the device.

### Azure Key Vault Integration

``` powershell title="bios-cert-keyvault.ps1"
# Authenticate to Azure
Connect-AzAccount

# Sign a single BIOS setting change via Key Vault
$cmd = Get-LnvSignedWmiCommand -Method SetBiosSetting `
    -VaultName "CorpKeyVault" -KeyName "BiosSigningKey" `
    -SettingName "WakeOnLAN" -SettingValue "Enable"
Submit-LnvBiosChange -Command $cmd

# Or process an entire INI config batch
Convert-LnvBiosConfigFile -ConfigFile "settings.ini" `
    -VaultName "CorpKeyVault" -KeyName "BiosSigningKey"
Submit-LnvBiosConfigFile -ConfigFile "SignedSettings.ini"
```

Install the optional GUI for certificate management:

``` powershell
Install-Script 'LnvBiosCertInterface'
```

## Getting Started

Install everything from an elevated PowerShell session:

``` powershell
# Install the three modules
Install-Module 'Lenovo.Client.Update'
Install-Module 'Lenovo.BIOS.Config'
Install-Module 'Lenovo.Bios.Certificates'

# Install the optional GUI tools
Install-Script 'ThinkBiosConfigUI'
Install-Script 'LnvBiosCertInterface'

# Verify: run your first update check
Get-LnvUpdate | Format-Table -Property Title, Category, ReleaseDate
```

## Resources

### Product Guides

- [Lenovo.Client.Scripting module](https://docs.lenovocdrt.com/guides/lcsm/lcsm_top/)
- [Think BIOS Config Tool V2 and Lenovo.BIOS.Config module](https://docs.lenovocdrt.com/guides/tbct_v2/tbct_v2_top/)
- [Lenovo BIOS Certificate Tool and Lenovo.Bios.Certificates module](https://docs.lenovocdrt.com/guides/lbct/)
- [Lenovo.Client.Update module](https://docs.lenovocdrt.com/guides/lcu/)

### Other Resources

- [PowerShell Gallery](https://www.powershellgallery.com/) — search "Lenovo"
- [ThinkDeploy Docs](https://docs.lenovocdrt.com)
- [ThinkDeploy Blog](https://blog.lenovocdrt.com)
- [Lenovo Forums](https://forums.lenovo.com) — Enterprise Client Management Board
