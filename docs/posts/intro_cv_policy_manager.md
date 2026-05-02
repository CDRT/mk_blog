---
date:
    created: 2026-05-02
authors:
    - Phil
categories:
    - "2026"
title: "Introducing: Commercial Vantage Policy Manager for Intune"
---

Commercial Vantage ships ADMX/ADML templates for configuring its policies, and the documented path for Intune is to either ingest those templates and configure each setting or create a custom template profile and add each individual policy as an OMA-URI setting. That works, but can be tedious. **Commercial Vantage Policy Manager** is a WPF-based PowerShell GUI that takes the same policies and deploys them as a single Custom OMA-URI profile through the Microsoft Graph API — no template ingestion required.
<!-- more -->

<video controls width="100%" preload="metadata">
  <source src="https://cdrt.github.io/mk_blog/img/2026/intro_cv_policy_manager/CVPolicyManager.mp4" type="video/mp4">
</video>

## Why an alternative?

Both approaches end up writing the same policy CSPs to the device. The difference is operational:

- **ADMX ingestion** — import the templates, then configure each policy individually in the Imported Administrative templates profile. If any new policies are introduced, you'll need to [replace the ADMX files](https://learn.microsoft.com/intune/device-configuration/settings-catalog/import-custom-admx-templates#replace-existing-admx-files) that were imported.
- **Custom OMA-URI** — one profile holds every configured policy as an explicit OMA-URI string. Easy to review, easy to export, easy to recreate in another tenant.

The Policy Manager builds the OMA-URI profile for you so you don't have to hand-author the URIs or copy/paste each value individually from our [docs](https://docs.lenovocdrt.com/guides/lcv/configuration/#policies).

## Requirements

- PowerShell **5.1** or higher
- .NET Framework **4.7.2** or later (shipped with Windows 10/11)
- The `Microsoft.Graph.Authentication` module:

    ``` powershell
    Install-Module Microsoft.Graph.Authentication -Scope CurrentUser
    ```

- Internet access to fetch the policy catalog (a local copy of `cv-policies.json` works as a fallback)
- An Entra ID account with permission to create Intune device configuration profiles

## Getting the tool

Install the tool from the PowerShell gallery

```powershell
Install-Script Invoke-CVPolicyManger
```

## The policy catalog

The list of policies, their OMA-URIs, data elements, and tree structure is **not** hardcoded in the script. It lives in `cv-policies.json` in the [CDRT/api](https://github.com/CDRT/api) repository. On startup the tool fetches the latest copy from:

```
https://raw.githubusercontent.com/CDRT/api/main/cv-policies.json
```

## Workflow

### 1. Sign in

Click **Sign In** in the header to authenticate to Microsoft Graph. Choose either interactive, app registration with certificate, or app registration with secret.

The tool uses the standard interactive flow exposed by `Microsoft.Graph.Authentication` and requests the `DeviceManagementConfiguration.ReadWrite.All` scope.

![Sign In](https://cdrt.github.io/mk_blog/img/2026/intro_cv_policy_manager/image2.jpg)

### 2. Apply the Recommended Baseline (optional)

The left navigation has a **Baselines** section with a *Recommended Baseline* entry. Selecting it shows the policies that make up the baseline and an **Apply Baseline** button that enables all of them in a single click.

The recommended baseline policies are:

| Policy | Purpose |
|---|---|
| Accept EULA | Suppresses the EULA prompt on first launch |
| Write Warranty to WMI | Exposes warranty data to WMI for inventory tools |
| Write Battery to WMI | Exposes battery health to WMI |
| Turn off Metrics Collection | Disables telemetry |
| Turn off Network | Disables the Network feature in Vantage |
| Turn off Sustainability | Hides the Sustainability page |
| Turn off Give Feedback | Hides the Give Feedback option |
| Turn off Run Once Task | Suppresses first-run popups |

These are the settings most enterprise environments end up configuring anyway, bundled so you don't have to click through each one.

### 3. Browse and configure

The **Policies** tree mirrors the category structure from the catalog. Selecting a category renders a card for every policy in that section. Each card has three states:

- **Not Configured** — policy is omitted from the resulting profile
- **Enabled** — policy is enabled; any data elements (text fields, numeric inputs, checkboxes, dropdowns) become editable
- **Disabled** — policy is explicitly set to disabled

Numeric data elements honor the `min`/`max` values from the catalog, so out-of-range values are blocked at input time rather than failing on the device.

The footer shows a running count of how many policies are currently configured.

### 4. Create the Intune profile

Once the policies are set, edit the profile name at the bottom of the window if you want to override the default and click **Create Profile**.

![Create Profile](https://cdrt.github.io/mk_blog/img/2026/intro_cv_policy_manager/image4.jpg)

The default name follows a convention designed for clean sorting in the Intune console:

```
Win - Custom - Lenovo Commercial Vantage - D - v1.0
```

- **Win** — target platform
- **Custom** — profile type (Custom OMA-URI Template)
- **Lenovo Commercial Vantage** — application area
- **D** — assignment scope (D = Devices, U = Users)
- **v1.0** — version

The Description field is intentionally left blank — all metadata is encoded in the name itself, since descriptions are easy to overlook and tend to drift out of date.

The tool POSTs a `windows10CustomConfiguration` to Graph with one OMA setting per configured policy. Not Configured policies are simply omitted from the payload. After creation the profile is visible in the Intune admin center under **Devices → Configuration**, ready for assignment.

![Intune Profile](https://cdrt.github.io/mk_blog/img/2026/intro_cv_policy_manager/image5.jpg)

!!! note
    The tool creates the profile but does not assign it. Assignment is left to the admin so it doesn't accidentally roll out to every device in the tenant.

## Logging

Every session writes to a date-stamped log file:

```
%ProgramData%\Lenovo\CVPolicyManager\CVPolicyManager_YYYYMMDD.log
```

The same output streams to the **Log** panel inside the GUI (toggle it from the header). Log levels are `INFO`, `WARN`, `ERROR`, and `SUCCESS` — useful for confirming the catalog source, the Graph authentication result, and any failures during profile creation.

![Logging](https://cdrt.github.io/mk_blog/img/2026/intro_cv_policy_manager/image6.jpg)

## Theming

The window picks up the OS-level light/dark preference on launch by reading `HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Themes\Personalize\AppsUseLightTheme`. The sun/moon button in the header toggles between themes if you want to override it for the current session.

## Summary

If you manage Commercial Vantage with Intune and you don't want to maintain ingested ADMX templates, the Policy Manager is a straightforward alternative:

- Fetches the policy catalog from [CDRT/api](https://github.com/CDRT/api), so it stays current as policies are added
- Builds a single Custom OMA-URI profile via Microsoft Graph
- One-click **Recommended Baseline** for typical enterprise settings
- Validates data element types and ranges before deployment
- Logs everything to `%ProgramData%\Lenovo\CVPolicyManager`
