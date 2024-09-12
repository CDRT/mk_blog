---
date:
    created: 2019-05-15
authors:
    - Joe
categories:
    - "2019"
title: Introducing the Manage Driver Packs Feature
---

## What is it?

This feature allows an admin to create a collection of the hardware drivers for a specified model in a format that can be imported into Microsoft System Center Configuration Manager (SCCM) or Microsoft Deployment Toolkit (MDT) to support OS Deployment.
<!-- more -->
## How is it different?

This feature presents a list of only hardware drivers for a specified model based on what is currently available on the Lenovo Support web site instead of basing search results on content ready for use with System Update or Thin Installer. **NOTE: Only Windows 10 and 11 is in scope for this feature.**

## Who would use this?

IT administrators that are performing custom imaging using SCCM and MDT always need to start with a set of hardware drivers specific to a model in order to get the best results. Any IT admin that wants to reduce the amount of time this process takes AND wants to use only the very latest drivers released will gain an advantage using this feature.

## Walkthrough

1. Launch Update Retriever
![Launch UR](https://cdrt.github.io/mk_blog/img/2019/ur_manage_driver_packs/launchur.png)

2. Select download location and model
![Step 1](https://cdrt.github.io/mk_blog/img/2019/ur_manage_driver_packs/step1.png)

3. Select specific drivers
![Step 2](https://cdrt.github.io/mk_blog/img/2019/ur_manage_driver_packs/step2.png)

4. Update Retriever downloads an extracts selected drivers
![Step 3](https://cdrt.github.io/mk_blog/img/2019/ur_manage_driver_packs/step3.png)

5. Now you have collection of source files for a driver package...
![Step 4](https://cdrt.github.io/mk_blog/img/2019/ur_manage_driver_packs/step4.png)

...and a CSV report file
![Step 2](https://cdrt.github.io/mk_blog/img/2019/ur_manage_driver_packs/csv.png)

## Q & A

**Why only Windows 10 and 11?**

Windows 7 driver content available from Lenovo Support web site does not always have the INF installable source files directly accessible. With Windows 10 and later, there is a requirement for all hardware drivers to be INF installable so we are able to achieve consistent results only with Windows 10 and later drivers.

**What Lenovo products are supported?**

This feature will be limited to ThinkPad, ThinkCentre and ThinkStation products launched in 2018 and going forward. Content for older products did not meet the packaging requirements necessary to ensure consistent results by this feature.

**Why is there a difference in the content normally offered by Update Retriever and the content available through this feature?**

The content that you would normally manage in an Update Retriever content is designed to be used by System Update, Thin Installer, etc. There is additional work that occurs to enable these tools to automate the installation of this content and therefore it takes longer for the updates to become available. This new feature simply relies on the packages available for download and manual installation which become available sooner.

**What is the difference between a driver pack created with this feature versus the “SCCM Driver Packs” available for download from Lenovo Support web site?**

There are a few differences:

1. The SCCM driver packs available for download are created and validated as a set. Due to the time required to produce and publish these there can sometimes be newer drivers on the Lenovo Support web site than what is in a pack. Using this feature in Update Retriever will provide the most current drivers available.

2. The SCCM driver packs contain all of the hardware drivers that could be needed for a model. Depending on the particular model you have, there may be drivers that are not needed in the pack. Using this feature in Update Retriever you can control exactly which drivers you include.

3. In order to leverage the most current drivers, the package that is published for manual download and install is being used by this feature. In order to support the manual installation there may be some files in the package that are not necessary for OS deployment using SCCM or MDT. These files are typically removed from SCCM driver packs. Note: if the source files are imported into SCCM or MDT, only the files referenced specifically by an INF will be imported so the effect is minimal.

**Are BIOS and firmware updates covered?**

No, this new feature is intended to support the OS deployment process of new drivers which only works with INF installable hardware drivers. Application updates and firmware updates cannot be included.
