---
date:
    created: 2024-04-24
    updated: 2024-11-11
authors:
    - Phil
categories:
    - "2024"
title: Arm-based Operating System Deployment
---

![Windows on ARM](https://cdrt.github.io/mk_blog/img/2024/arm_osd/image.jpg)

Operating System Deployment [support for Windows 11 ARM 64](https://learn.microsoft.com/mem/configmgr/core/plan-design/changes/whats-new-in-version-2403#os-deployment) devices is now supported in the 2403 release of Configuration Manager. This guide will demonstrate how to deploy Windows 11 to the Qualcomm equipped ThinkPad X13s and T14s Gen 6.
<!-- more -->

## ThinkPad X13s

### ADK

At the time of initially publishing this article, I had installed the September 2023 [ADK (10.1.25398.1)](https://learn.microsoft.com/windows-hardware/get-started/adk-install#download-the-adk-101253981-september-2023) and the Windows PE add-on. This version is required for Windows ARM deployments. If you need to update the ADK from a previous version, do so before updating Configuration Manager. This will allow the default boot images to be automatically updated to the latest Win PE version.

### Site Update

Update 2403 is an in-console update. If you're upgrading, make sure to go through the install [checklist](https://learn.microsoft.com/mem/configmgr/core/servers/manage/checklist-for-installing-update-2403) before applying the update. If everything goes smoothly, you'll now see a new boot image in the console.

![Boot image](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image1.jpg)

![Boot image properties](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image2.jpg)

### PXE Configuration

During testing, the Distribution Point in my lab was configured for SCCM's PXE responder service (WDS-less). However, if you're going to deploy an ARM-based device, WDS PXE is the only way to do it (for now). Rather than messing with my existing one, I went ahead and stood up another DP in my lab and configured it for WDS PXE.

![PXE Configuration](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image3.jpg)

### Drivers

Next will be adding the right drivers to the boot image. When I initially PXE booted the X13s using a USB-C to Ethernet adapter, I successfully got into WinPE, but there was no touchpad/trackpoint or keyboard support so that left me dead in the water. At the time of this writing, the latest [SCCM pack](https://pcsupport.lenovo.com/products/laptops-and-netbooks/thinkpad-x-series-laptops/thinkpad-x13s-type-21bx-21by/downloads/ds556992-sccm-package-for-windows-11-arm-version-21h2-thinkpad-x13s?category=Enterprise%20Management) for X13s was released on April 5. You'll want to download the Windows 11 22H2 pack and extract the contents to your driver share.

In the console, go to **Software Library > Drivers** and click **Import Driver** in the ribbon bar to start the driver import wizard.

#### Boot Image

We're only going to import the **Boot Critical** drivers. Referencing the SCCM pack ReadMe, the Build ID for these drivers is **N3HQC12W** and can be found under the Chipset directory. The path to this directory is what you'll specify in the wizard to import.

![Boot critical drivers](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image4.jpg)

Wait several minutes for the drivers (52 total) to be imported to the driver catalog. Once that clears, keep the defaults checked and assign the drivers a category, optional of course. I created a **X13s** and **WinPE** category for organization purposes.

![Import drivers](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image5.jpg)

Skip to select packages to add the drivers to.

On the **Select drivers to include in the boot image**, tick the box beside the **Boot image (arm64)** and click next, which will then present you with this prompt

![Dialog](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image6.jpg)

Click Yes and you'll be greeted with one more

![Dialog](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image7.jpg)

Click next all the way through to complete adding the drivers to your boot image.

If you pull up the boot image properties and click on the Drivers tab, you should see all of the injected drivers

![Injected drivers](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image8.jpg)

With the properties still open, enable command line support under the **Customization** tab and add any other **Optional Components**, such as PowerShell support. Update Distribution Points and your boot image is ARM ready.

#### Driver Package

I tested both ways for driver installation. A [Driver Package](https://learn.microsoft.com/mem/configmgr/osd/get-started/manage-drivers#driver-packages) and a legacy Package containing the extracted drivers. I prefer the latter, so in the console, go to **Software Library > Packages** and create a new Package. Give it a name, tick the box beside **This package contains source files**, and point it to the path on your share where you extracted the drivers. Click the radio button beside **Do not create a program** and finish out the wizard. Distribute the Package to your Distribution Points.

### Windows 11 ARM Media

Download the Windows 11 ARM ISO and add the install.wim under **Software Library > Operating System Images**. This is 24H2 Enterprise

![ARM Media](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image9.jpg)

### Task Sequence

Create a new Task Sequence, selecting the option to **Install an existing image package**. Name it and choose the Arm boot image

![Task Sequence](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image10.jpg)

Choose the Windows 11 Arm image package and proceed through the Task Sequence creation wizard.

Once complete, edit the Task Sequence and disable the default step to apply drivers.

- Create a new group titled **Drivers**. Under that group, add a **Download Package Content** step and add the driver package created earlier. I'll tick the radio button to save the content to a custom path of **%_SMSTSMDataPath%\Drivers**

!!! info ""
    More info on TS variables can be found here <https://learn.microsoft.com/mem/configmgr/osd/understand/task-sequence-variables#SMSTSMDataPath>

![Task Sequence](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image11.jpg)

- Add a **Run Command Line** step that calls Dism to recursively install the drivers.

```cmd
DISM.exe /Image:%OSDTargetSystemDrive%\ /Add-Driver /Driver:%_SMSTSMDataPath%\Drivers /Recurse /LogPath:%_SMSTSLogPath%\DISM.log
```

![Task Sequence](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image12.jpg)

Now deploy the Task Sequence to the Unknown Computers device collection.

### Experience

Firing up the X13s and F12'ing to the boot menu to PXE

![Boot Menu](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image13.jpg)

Boot image incoming!

![Downloading boot image](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image14.jpg)

Unfortunately, we have a resolution problem which I can't seem to get sorted. I've even tried dumping the entire driver pack in the boot image but that didn't fix it. On the bright side, you can kick off your Task Sequence.

![Task Sequence](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image15.jpg)

Successful deployment with a clean Device Manager!

![Success](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image16.jpg)

![Clean Device Manager](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image17.jpg)

## ThinkPad T14s Gen 6

### ADK

At the time of updating this article, my ConfigMgr environment is at version [2403](https://learn.microsoft.com/mem/configmgr/core/plan-design/changes/whats-new-in-version-2403) with [ADK 10.1.26100 (May 2024)](https://learn.microsoft.com/windows-hardware/get-started/adk-install#download-the-adk-101261001-may-2024) installed.

### Boot Image

Looking at the ARM64 boot image properties, the version is **10.0.26100**.

![BootImageProps](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image18.jpg)

Switching to the **Drivers** tab, you can see no additional drivers have been injected.

![Drivers](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image19.jpg)

### PXE Boot

On my T14s Gen 6, I'll activate System Deployment Boot Mode at the boot menu and proceed to PXE booting

![Success](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image20.jpg)

Here, I can see the ARM64 boot image downloading (package ID matches)

![Success](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image21.jpg)

### Task Sequence

We've now entered PE and can continue to choose a task sequence. Remember, no drivers were added to the boot image so the fact that we can see the wizard confirms inbox driver support for this model.

!!! info ""
    The SCCM pack for T14s Gen 6 can be downloaded [here](https://pcsupport.lenovo.com/products/laptops-and-netbooks/thinkpad-t-series-laptops/thinkpad-t14s-gen-6-type-21n1-21n2/21n2/downloads/ds570049-sccm-package-for-windows-11-arm-version-23h2-thinkpad-thinkpad-t14s-gen-6-type-21n1-21n2?category=Enterprise%20Management)
    
    The HSA pack can be downloaded [here](https://pcsupport.lenovo.com/products/laptops-and-netbooks/thinkpad-t-series-laptops/thinkpad-t14s-gen-6-type-21n1-21n2/21n2/downloads/ds570048-hsa-package-for-windows-11-arm-version-23h2-thinkpad-thinkpad-t14s-gen-6-type-21n1-21n2?category=Enterprise%20Management)

![Success](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image22.jpg)

![Success](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image23.jpg)

### End Results

Once OSD completes, I can see a clean device manager.

![Success](https://cdrt.github.io/mk_blog/img\2024\arm_osd\image24.jpg)