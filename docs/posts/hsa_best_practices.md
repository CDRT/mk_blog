---
date:
    created: 2026-02-26
    updated: 2026-02-26
authors:
    - Joe
categories:
    - "2026"
title: HSA Best Practices Guide
---

![Manage PC](https://cdrt.github.io/mk_blog/img/2026/hsa_best_practices/manage_pc.png){ width="360" }

Hardware Support Apps (HSAs) are a core part of the modern Windows driver model — UWP/MSIX apps published by OEMs or IHVs that extend a base driver, typically delivered via the Microsoft Store.  Common examples include Intel Graphics Command Center, AMD Software, Realtek Audio Console, and Lenovo hardware control apps.  They are a fundamental piece of the **Declarative, Componentized, Hardware Support Apps (DCH)** driver architecture introduced in Windows 10 1809.

Understanding how HSAs are designed to work — and where enterprise management intersects with that design — is critical to deploying and maintaining a clean, supportable Windows environment.
<!-- more -->

## What HSAs Are

Hardware Support Apps are:

- **UWP / MSIX apps** published by OEMs or IHVs
- Designed to **extend a base driver** (which is typically delivered via Windows Update, other patching tools or as part of an OS deployment)
- **Store-delivered**, not bundled in traditional hardware driver packages

Because HSAs are paired to their base drivers at the INF level, the driver knows which app to request and the Store knows which version to deliver.  This pairing is intentional — and any management approach that works against it will be fragile and difficult to support.

## The Golden Rule

> **Do not try to "install HSAs" during imaging.**
> Let Windows and the Microsoft Store install them **after the base driver is present**.

Any strategy that fights this model creates unnecessary complexity and increases the risk of broken driver/app pairings over time. The one exception to this is the situation where devices must be deployed in a completely "air-gapped" environment. This scenario is why we provide HSA packs that contain the files necessary to side-load the apps during a deployment process.

!!! note
    See the [Hardware Support Apps without Microsoft Store](https://blog.lenovocdrt.com/hardware-support-apps-without-microsoft-store/) article for Lenovo-specific guidance on using HSA Packs with the Install-HSA.ps1 script.

## Recommended Deployment Model

### Best Practice: Let Windows Update + Microsoft Store Handle HSAs Automatically

**How it works**

1. Bare-metal Windows 11 deployment
2. Apply DCH base drivers from an SCCM driver pack or other collection of hardware drivers only
3. At first launch, Windows automatically applies drivers to matching hardware
4. Microsoft Store installs the associated HSA based on the drivers applied

**Why this is the preferred model**

- Fully supported by Microsoft
- OEM- and IHV-validated pairing of driver and app
- Automatic updates without admin intervention
- Works cleanly with Autopilot and Configuration Manager

**What you must ensure**

- Microsoft Store is **not disabled**
- Store auto-update is **allowed**
- Device has internet access post-OOBE as HSAs will install at first boot

This is the model Microsoft expects enterprises to use.

---

## Recommended Store Policies

If you restrict the Store, you still should not sideload HSAs manually.  The correct approach is to restrict the **user experience** without disabling the underlying Store services. Getting Store policy right is the most common point of failure in enterprise HSA deployments.  The key principle is:

> **Do NOT disable the Microsoft Store service.**
> Restrict **user interaction**, not **Store functionality**.

#### 1. Turn Off the Microsoft Store App (UI Only)

This blocks user access **without disabling background installs**.

**Intune (Settings Catalog)**

```
Administrative Templates
└─ Windows Components
   └─ Store
      └─ Turn off the Store application = Enabled
```

**Group Policy**

```
Computer Configuration
└─ Administrative Templates
   └─ Windows Components
      └─ Store
         └─ Turn off the Store application = Enabled
```

Result: Store app UI is blocked, Store services still run, HSAs still install and update.

#### 2. Allow Store Apps to Update Automatically

HSAs rely on Store auto-update, even if users cannot open the Store.

**Intune**

```
Administrative Templates
└─ Windows Components
   └─ Store
      └─ Turn off Automatic Download and Install of updates = Disabled
```

**Group Policy**

```
Turn off Automatic Download and Install of updates = Disabled
```

#### 3. Do Not Disable Required Store Services

Ensure these services are **not disabled** by security baselines or hardening scripts:

| Service | Startup |
|---|---|
| Microsoft Store Install Service (InstallService) | Manual / Trigger Start |
| Windows Update (wuauserv) | Automatic |
| Background Intelligent Transfer Service (BITS) | Automatic |
| Delivery Optimization (DoSvc) | Automatic |

!!! note
    HSAs **will not install** if InstallService is disabled.

### Recommendations

#### Use Device Context, Not User Context

HSAs are designed to install per-device in system context.  Ensure all policies apply to **Computer**, not **User**.

#### Do Not Remove These Store System Apps

- Microsoft.WindowsStore
- Microsoft.StorePurchaseApp
- Microsoft.DesktopAppInstaller

Removing these system apps will break HSA installation and update flows.

### Result

- HSAs still auto-install and auto-update
- Users do not see the consumer Store experience
- Security and compliance requirements are satisfied

## Governance and Lifecycle Management

### Updates

- HSAs update via **Microsoft Store**
- Drivers update via **Windows Update** or Lenovo tools
- They are intentionally decoupled — you do not need to synchronize versions manually

## Summary

The minimum safe configuration for a managed enterprise environment:

✅ Use **DCH drivers only**

✅ Let **Windows Update install drivers**

✅ Allow **Microsoft Store auto-install** of HSAs

✅ Restrict Store UX, not functionality

✅ Avoid task-sequence installation of HSAs

✅ Only sideload HSAs in offline scenarios

✅ **Turn off Store application = Enabled** (blocks UI, not services)

✅ **Auto app updates = Allowed**

✅ **Store services enabled**

✅ **No Store app removal**

✅ **Computer-level policy**

This gives you maximum compatibility, minimal imaging complexity, and a clean supportable configuration.
