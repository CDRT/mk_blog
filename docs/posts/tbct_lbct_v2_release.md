---
date:
    created: 2025-11-04
authors:
    - Devin
    - Joe
categories:
    - "2025"
title: Introducing Think BIOS Config Tool V2 and Lenovo BIOS Certificate Tool V2
---

![Tools Banner](https://cdrt.github.io/mk_blog/img/2025/tbct_lbct_v2_release/tbct_lbct_title.png)

We're excited to announce the release of two powerful new PowerShell-based tools for managing BIOS settings on Lenovo commercial PCs: **Think BIOS Config Tool V2** and **Lenovo BIOS Certificate Tool V2**. These tools represent a complete rewrite of their predecessors, bringing modern PowerShell capabilities, enhanced user interfaces, and seamless integration with Microsoft Intune.
<!-- more -->

## What's New?

The Think BIOS Config Tool been completely rebuilt from the ground up as a PowerShell-based solution with WPF graphical interface, replacing the older HTA-based version. It leverages the new Lenovo.BIOS.Config module which can also be used in your own scripts. This new version also includes integration with Intune.

The Lenovo BIOS Certificate Tool has been updated with a new UI and the Lenovo.BIOS.Certificates module has been updated to include support for Azure Key Vault for storage of private keys used in signing the settings change commands.

This modernization brings significant improvements in functionality, usability, and deployment options.

!!! note ""
    Documentation for these solutions is available on the CDRT Docs site:

    - [Think BIOS Config Tool V2](https://docs.lenovocdrt.com/guides/tbct_v2/tbct_v2_top/)
    - [Lenovo BIOS Certificate Tool V2](https://docs.lenovocdrt.com/guides/lbct/)

### Think BIOS Config Tool V2

The Think BIOS Config Tool V2 (`ThinkBIOSConfigUI.ps1`) is a comprehensive solution for managing BIOS settings on Lenovo Think devices. Built on the `Lenovo.BIOS.Config` PowerShell module, it provides both GUI and command-line interfaces for complete BIOS configuration management.

!!! note ""
    **This solution replaces the older Think BIOS Config Tool which was implemented as an HTA.** Archived documentation for the HTA version is still available here: [Think BIOS Config Tool - HTA](https://docs.lenovocdrt.com/guides/tbct/tbct_top/)

    Previously created INI files from the HTA version which contain an encrypted password are not compatible with the new Think BIOS Config Tool V2 due to changes in encryption methods. Please recreate the INI files using the new tool.


![Think BIOS Config Tool V2](https://cdrt.github.io/mk_docs/img/guides/tbct_v2/initial-settings.png)

#### Key Features

**Interactive Settings Management**

- Visual two-column settings display with real-time change indicators
- ComboBox controls for analog settings and TextBox controls for time/date/boot order
- Unsaved changes are highlighted in red for easy identification
- Save or revert changes with a single click

**Configuration Export/Import**

- Export current BIOS settings to INI files
- Import settings from INI files with password support
- Encrypted password storage using configurable passphrases
- Password-change file creation for remote deployments

**Factory and Custom Defaults**

- Reset to factory default settings
- Save custom default profiles
- Restore to saved custom defaults
- Manage multiple configuration profiles

**Microsoft Intune Integration**

- Create Win32 App packages automatically
- Generate Proactive Remediation scripts
- Direct upload to Intune via Microsoft Graph API
- Configurable detection rules and package metadata

**Enhanced Security**

- Supervisor password management
- Fingerprint data clearing
- Password change file generation
- Encrypted password storage in INI files

#### Installation

The tool is available from the PowerShell Gallery:

```PowerShell
# Install the UI script
Install-Script 'ThinkBiosConfigUI'

# Install the required module
Install-Module 'Lenovo.BIOS.Config'

# (Optional) Install Microsoft Graph Authnetication module for Intune integration
Install-Module Microsoft.Graph.Authentication -Scope CurrentUser -Force
```

#### Quick Start

Launch the GUI with administrator privileges:

```PowerShell
ThinkBIOSConfigUI.ps1
```

### Lenovo BIOS Certificate Tool V2

The Lenovo BIOS Certificate Tool V2 (`LnvBIOSCertificateTool.ps1`) complements the Think BIOS Config Tool by enabling certificate-based BIOS authentication. This eliminates the need to exchange passwords in plain text when scripting BIOS settings changes on supported Lenovo devices (2022 ThinkPad models and later).

![Lenovo BIOS Certificate Tool](https://cdrt.github.io/mk_blog/img/2023/cert_based_bios_authentication/installcertificatewindow.png)

#### Key Features

**Certificate Management**

- Install signing certificates to replace supervisor passwords
- Support for both PEM and DER certificate formats
- Dual certificate support for 2025 and later models (Supervisor and System Management)
- Certificate update and removal capabilities

**Signed Command Generation**

- Generate cryptographically signed WMI commands
- Support for all WMI BIOS methods
- Visual setting selection with WMI data loading
- Copy signed commands directly to clipboard

**Configuration File Conversion**

- Convert Think BIOS Config Tool INI files to signed commands
- Batch conversion of multiple settings
- Preserve configuration file structure

**Azure Key Vault Integration**

- Sign commands using keys stored in Azure Key Vault
- Eliminates need to distribute private keys
- Enterprise-grade key management
- Support for both local files and Azure integration

#### Certificate-Based Authentication Workflow

The certificate-based approach follows these steps:

1. **Create a Code Signing Certificate**: Generate a public/private key pair using OpenSSL or your PKI infrastructure
2. **Provision Devices**: Install the public certificate on target devices
3. **Generate Signed Commands**: Create cryptographically signed WMI commands using your private key
4. **Apply Commands**: Execute signed commands on target devices without requiring passwords
5. **Repeat as Needed**: Continue generating and applying signed commands for configuration changes

#### Supported Methods

The tool supports signing commands for the following WMI methods:

- **SetBiosSetting** - Change individual BIOS settings
- **SaveBiosSettings** - Commit settings changes
- **ClearBiosCertificate** - Remove certificate authentication
- **ChangeBiosCertificateToPassword** - Switch back to password authentication
- **UpdateBiosCertificate** - Replace an existing certificate
- **LoadDefaultSettings** - Reset to default settings
- **LoadFactoryDefaultSettings** - Reset to factory defaults
- **SetFunctionRequest** - Execute specific functions (TPM clear, fingerprint reset, etc.)
- **LoadCustomDefaultSettings** - Restore custom defaults
- **SaveCustomDefaultSettings** - Save current settings as custom defaults

#### Installation

```PowerShell
# Install the required module
Install-Module 'Lenovo.BIOS.Certificates'

# (Optional) For Azure Key Vault integration
Install-Module Az.Accounts
Install-Module Az.KeyVault
```

## Perfect Together

While each tool is powerful on its own, they're designed to work together seamlessly. The Think BIOS Config Tool can generate INI files with configuration settings, and the Lenovo BIOS Certificate Tool can convert those files into signed commands for password-less deployment.

**Example Workflow:**

1. Use Think BIOS Config Tool to define your desired BIOS configuration
2. Export settings to an INI file
3. Use Lenovo BIOS Certificate Tool to convert the INI to signed commands
4. Deploy signed commands via Intune, ConfigMgr, or your preferred deployment method
5. Apply settings without exposing supervisor passwords

## PowerShell Module Capabilities

Both tools are built on robust PowerShell modules that can be used independently for scripting and automation:

### Lenovo.BIOS.Config Module (v1.0.2)

Key cmdlets include:

- `Initialize-LnvThinkBiosConfig` - Gather WMI BIOS data
- `Show-LnvWmiSettings` - Display current settings
- `Export-LnvWmiSettings` - Export settings to INI
- `Import-LnvWmiSettings` - Apply settings from INI
- `Set-LnvWmiSetting` - Change individual settings
- `Export-LnvPasswordChangeFile` - Create password change files
- `Clear-LnvSupervisorPassword` - Remove supervisor password
- `Clear-LnvFingerprintData` - Clear fingerprint data

### Lenovo.BIOS.Certificates Module (v1.0.8)

Key cmdlets include:

- `Set-LnvBiosCertificate` - Install a certificate
- `Get-LnvSignedWmiCommand` - Generate signed commands
- `Submit-LnvBiosChange` - Apply signed commands
- `Convert-LnvBiosConfigFile` - Convert INI to signed commands
- `Get-LnvUnlockCode` - Generate BIOS unlock codes

This module supports Azure Key Vault integration for enterprise key management scenarios.

## Use Cases

These tools excel in several common enterprise scenarios:

**Initial Provisioning**

- Configure BIOS settings during imaging or autopilot
- Set consistent security policies across fleets
- Configure boot order and hardware settings

**Ongoing Management**

- Push BIOS updates via Intune Proactive Remediations
- Respond to security requirements with setting changes
- Manage settings without physical access to devices

**Security Hardening**

- Deploy certificate-based authentication to eliminate password sharing
- Centralize BIOS security management
- Maintain audit trails of configuration changes

**Migration Scenarios**

- Standardize settings across different device models
- Convert legacy password-based configs to certificate-based
- Document and replicate configurations

## Documentation and Support

Comprehensive documentation is available for both tools:

- **Think BIOS Config Tool V2**: [https://docs.lenovocdrt.com/guides/tbct_v2/tbct_v2_top/](https://docs.lenovocdrt.com/guides/tbct_v2/tbct_v2_top/)
- **Lenovo BIOS Certificate Tool**: [https://docs.lenovocdrt.com/guides/lbct/](https://docs.lenovocdrt.com/guides/lbct/)
- **Certificate-Based Authentication Guide**: [https://docs.lenovocdrt.com/guides/tbct_v2/cert_based_bios_authentication/](https://docs.lenovocdrt.com/guides/tbct_v2/cert_based_bios_authentication/)
- **Module Reference Guides**:
    - [Lenovo.BIOS.Config Module Reference](https://docs.lenovocdrt.com/guides/tbct_v2/tbc_module_reference/)
    - [Lenovo.BIOS.Certificates Module Reference](https://docs.lenovocdrt.com/guides/lbct/lbc_module_reference/)

## Prerequisites and Requirements

**Think BIOS Config Tool V2**

- Windows with PowerShell 5.1+ or PowerShell Core
- Administrative privileges
- For Intune features: Microsoft Graph modules and appropriate permissions
- NOTE: ThinkCentre desktops are not currently supported due to incompatible WMI BIOS Interface implementation.

**Lenovo BIOS Certificate Tool V2**

- Windows with PowerShell 5.1+ or PowerShell Core
- Administrative privileges
- Lenovo ThinkPad (2022+), ThinkCentre (2020+), or ThinkStation (2020+) with certificate support
- For Azure Key Vault: Az.Accounts and Az.KeyVault modules

## Getting Started

1. **Install the modules** from PowerShell Gallery
2. **Review the documentation** to understand capabilities
3. **Test on a single device** to verify compatibility
4. **Create your configuration** using the GUI or cmdlets
5. **Deploy at scale** using your preferred management platform

For certificate-based authentication, you'll also need to:

1. **Generate or obtain** a code signing certificate and private key
2. **Test provisioning** on a pilot device
3. **Generate signed commands** for your desired settings
4. **Validate the process** before wider deployment

## Conclusion

The Think BIOS Config Tool V2 and Lenovo BIOS Certificate Tool V2 represent a significant advancement in BIOS management capabilities for Lenovo commercial devices. Whether you're managing a handful of devices or thousands, these tools provide the flexibility, security, and integration options needed for modern IT environments.

The combination of intuitive graphical interfaces, powerful PowerShell modules, and seamless Intune integration makes it easier than ever to maintain consistent, secure BIOS configurations across your Lenovo fleet.

Download and install the tools today from the PowerShell Gallery to start streamlining your BIOS management workflows!

---

*For questions, issues, or feature requests, please visit the [Enterprise Client Management](https://forums.lenovo.com/t5/Enterprise-Management-Board/bd-p/sa01_eg) or consult the documentation links provided above.*
