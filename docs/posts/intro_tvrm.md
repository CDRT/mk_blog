---
date:
    created: 2026-06-02
authors:
    - Joe
categories:
    - "2026"
title: "Introducing ThinkVantage Repository Manager"
---

Managing a local Lenovo update repository has traditionally meant Update Retriever — a tool that gets the job done but leaves little room for automation or operational visibility. ThinkVantage Repository Manager (TVRM) is its modern replacement: a WPF GUI backed by a full PowerShell module that supports multiple repositories, per-model targeting, and a per-repository audit log that tracks every operation by user and timestamp.

<!-- more -->

## Why Replace Update Retriever?

Update Retriever works, but it was designed for an era before scripted management pipelines. Two gaps matter most in modern fleet management:

**Automation.** Update Retriever has no scriptable interface. Every repository operation — searching, downloading, promoting packages — requires the GUI. TVRM ships the `Lenovo.Client.RepositoryManager` PowerShell module as its backend. Every action the GUI performs maps directly to a cmdlet, so you can drive the entire repository lifecycle from a scheduled task, a CI pipeline, or a maintenance script without any manual steps.

**Auditability.** Update Retriever gives you no record of who changed what or when. TVRM writes a daily structured audit log (`audit_YYYY-MM-DD.log`) to a `logs` subfolder inside each repository. Every download, deletion, status promotion, and signature failure is stamped with a timestamp, the Windows username, and a human-readable message. That log is queryable with `Get-LnvRMAuditLog` and pipeable to anything that accepts objects.

These two capabilities — scriptable automation and structured audit history — are the primary reasons to move off Update Retriever.

## Getting TVRM

The tool ships as a single PowerShell script (`ThinkVantageRepositoryManager.ps1`) that wraps the `Lenovo.Client.RepositoryManager` module. The script handles module resolution automatically: it checks for an installed version first, then falls back to the PowerShell Gallery, then to a bundled copy in the same folder.

To install the GUI script from the PowerShell Gallery manually:

```powershell
Install-Script -Name ThinkVantageRepositoryManager -Scope CurrentUser
```

If you only want the module, it can also be installed from the PowerShell Gallery:

```powershell
Install-Module -Name Lenovo.Client.RepositoryManager -Scope CurrentUser
```

Or download the [script and module bundle](https://download.lenovo.com/cdrt/tools/Lenovo.Client.RepositoryManager_1.0.1.zip) for environments without Gallery access.

Launch the GUI from an elevated PowerShell session:

```powershell
.\ThinkVantageRepositoryManager.ps1
```

**Requirements:** Windows 11, PowerShell 5.1, internet access to Lenovo download servers.

## The GUI

The interface has three areas: the header bar, a tabbed content area, and a status bar.

![ThinkVantage Repository Manager launch screen](https://cdrt.github.io/mk_blog/img/2026/intro_tvrm/tvrm_launch.png)

The header holds a theme toggle (light/dark, auto-detected from Windows on launch) and the Settings gear. The status bar at the bottom shows the active repository name, mode (Full or Hybrid), configured model count, and a progress bar during downloads.

### First-Time Setup

Before searching, configure at least one repository and one model in Settings.

**Repositories** — Give it a name, a local folder path, and choose a mode:

- **Full**: Downloads the complete package — installer, descriptor XML, readme, and external detection routine files. Use for repositories that serve as a local deployment source for Commercial Vantage, Thin Installer, or Lenovo.Client.Update scripts.
- **Hybrid**: Downloads metadata only (descriptor XML and readme). Client devices fetch installers directly from the Lenovo CDN at install time. Reduces repository storage when bandwidth at download time is not a concern. Use this with Commercial Vantage, it is not supported by Thin Installer.

**Models** — Add each model by its 4-character Machine Type code (e.g., `21NT` for ThinkPad X1 Carbon Gen 13), a friendly name, and target OS. If you don't have the Machine Type handy, run this on the device:

```powershell
(Get-WmiObject Win32_ComputerSystemProduct).Name.Substring(0,4)
```

### Searching and Downloading Updates

The **Search Updates** tab queries Lenovo's online catalogs for all configured models in parallel. Results show title, version, type, severity, reboot behavior, release date, size, and which models each package applies to. Packages already in the active repository are highlighted green — selecting them again skips the download and instead syncs any new model associations into `database.xml`.

To download, select one or more rows, set the initial **Status** to `Test` or `Active`, and click **Download Selected**. TVRM verifies the SHA-256 checksum and Lenovo digital signature on each installer and descriptor XML. Any package that fails signature verification is rejected and logged to the audit trail — it never touches the repository.

![Search results](https://cdrt.github.io/mk_blog/img/2026/intro_tvrm/tvrm_search_results.png)

### Managing Repository Contents

The **Repository** tab shows everything in the active repository. Filter by title keyword, status, severity, or model. Superseded packages appear dimmed and italic with the superseding package ID shown — useful for identifying cleanup candidates.

Status promotion is handled by a **Test / Active** toggle on the selected row. The change writes to `database.xml` immediately. This Test → Active workflow gives you a validation gate: download new packages at Test, validate on a pilot group, then promote to Active so the broader fleet picks them up.

## Scripted Workflows With the Module

Every GUI operation has a cmdlet equivalent. Here are the patterns that matter most:

### Initial Repository Population

```powershell
New-LnvRMRepository -Path 'D:\LenovoUpdates\Production' -Name 'Production' -SetActive

Add-LnvRMModel -MachineType '21NT' -FriendlyName 'ThinkPad X1 Carbon Gen 13' -OS 'Windows 11'
Add-LnvRMModel -MachineType '21AH' -FriendlyName 'ThinkPad T14s Gen 3' -OS 'Windows 11'

$updates = Search-LnvRMUpdate | Where-Object { $_.Severity -in 'Critical', 'Recommended' }
$result  = Save-LnvRMUpdate -Update $updates -Status 'Test'

Write-Host "Downloaded: $($result.Downloaded.Count)"
Write-Host "Signature failures: $($result.SignatureSkipped.Count)"
```

### Bulk Status Promotion

```powershell
Get-LnvRMRepoContent -Status 'Test' | ForEach-Object {
    Set-LnvRMUpdateStatus -PackageID $_.PackageID -Status 'Active'
}
```

### Removing Superseded Packages

```powershell
Get-LnvRMRepoContent | Where-Object { $_.IsSuperseded } |
    ForEach-Object { Remove-LnvRMUpdate -PackageID $_.PackageID }
```

### Reviewing the Audit Log

```powershell
# Last 20 entries
Get-LnvRMAuditLog -Last 20 | Format-Table Timestamp, User, Action, Message -AutoSize

# Surface any signature failures
Get-LnvRMAuditLog | Where-Object { $_.Action -eq 'SIGNATURE_FAIL' }
```

The audit log records these action codes: `REPO_CREATE`, `REPO_CONFIG`, `DOWNLOAD`, `DELETE`, `STATUS_CHANGE`, `SIGNATURE_FAIL`, `EXTERNAL_FAIL`, and `SYNC`. Every entry carries the Windows username, so you have a clear record of who made each change — useful when multiple admins share a repository path.

!!! note
    All module cmdlets that modify state support `-WhatIf` and `-Confirm`, so you can preview destructive operations before committing.

## BITS vs. WebClient Downloads

The **DOWNLOAD METHOD** setting in Settings (or `Set-LnvRMPreference -Name 'DownloadMethod' -Value 'BITS'`) controls the transfer mechanism. **WebClient** is the default and fastest option for stable connections. **BITS** resumes interrupted transfers and runs at low priority — prefer it in environments with unreliable connectivity or where BranchCache/LEDBAT caching is in play.

## Full Module Reference

The complete cmdlet reference is available at [docs.lenovocdrt.com](https://docs.lenovocdrt.com/guides/tvrm/module_reference).

| Cmdlet | Purpose |
| --- | --- |
| `New-LnvRMRepository` | Create a new local update repository |
| `Get-LnvRMRepository` | List registered repositories |
| `Set-LnvRMRepository` | Modify settings or switch the active repository |
| `Remove-LnvRMRepository` | Unregister a repository |
| `Add-LnvRMModel` | Register a model for update searches |
| `Get-LnvRMModel` | List configured models |
| `Remove-LnvRMModel` | Remove a model |
| `Search-LnvRMUpdate` | Query Lenovo catalogs for available updates |
| `Save-LnvRMUpdate` | Download updates to the active repository |
| `Get-LnvRMRepoContent` | List packages in the repository |
| `Set-LnvRMUpdateStatus` | Promote or demote between Test and Active |
| `Remove-LnvRMUpdate` | Remove a package from the repository |
| `Sync-LnvRMSupportedSystem` | Add missing model associations to existing packages |
| `Get-LnvRMAuditLog` | Read repository audit log entries |
| `Get-LnvRMPreference` / `Set-LnvRMPreference` | Read/write application preferences |

Have questions? Visit the [Enterprise Client Management Forum](https://forums.lenovo.com/t5/Enterprise-Client-Management/bd-p/sa01_eg).
