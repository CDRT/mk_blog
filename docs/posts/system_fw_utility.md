---
date:
    created: 2025-03-26
authors:
    - Phil
categories:
    - "2025"
title: What Is The System Firmware Update Utility?
---

You may start seeing a new update offered titled **System Firmware Update Utility**. What exactly is this and what does it fix?

<!-- more -->

Starting with the X1 Carbon Gen 13, and eventually common for all ThinkPad 2025 products, this System Firmware Update Utility now bundles:

- System Firmware
- UEFI BIOS
- Embedded Controller Firmware (ECP)
- Intel Management Engine Firmware
- Power Delivery Firmware

This can be found in the ReadMe

![ReadMe](https://cdrt.github.io/mk_blog/img/2025/system_fw_utility/image1.jpg)

Here is what is presented in Commercial Vantage.

![CV](https://cdrt.github.io/mk_blog/img/2025/system_fw_utility/image2.jpg)

Inspecting the package XML descriptor, we can see these updates are a RebootType 5 (reboot delayed) and PackageType 4 (Firmware).

![XML](https://cdrt.github.io/mk_blog/img/2025/system_fw_utility/image3.jpg)

The BIOS update utility has been revamped to now show a comparison between the currently installed firmware versions and the newer versions.

![WINUPTP](https://cdrt.github.io/mk_blog/img/2025/system_fw_utility/image6.jpg)

The end-user experience shows a counter and which firmware is being updated now, which is helpful.

![UI](https://cdrt.github.io/mk_blog/img/2025/system_fw_utility/image4.jpg)

The System Firmware Version can be found in the BIOS Setup Main page. It is a version number representing the collection of **UEFI + EC + ME + PD** firmware.

![BIOS](https://cdrt.github.io/mk_blog/img/2025/system_fw_utility/image5.jpg)

While this is a great improvement in update delivery and reduces the amount of separate updates for a given device, with it comes implications. There will be the challenge of having multiple versions of different updates within the package that also has its own version number. If a future update is released that keeps the same version of BIOS but includes a newer version of the Intel MEFW, what would this look like if you're using the **Get-LnvAvailableBiosVersion** cmdlet from our [Lenovo Client Scripting module](https://docs.lenovocdrt.com/guides/lcsm/lcsm_top/)?

On the bright side, this should make patching a bit less painful.