---
date:
    created: 2020-12-10
authors:
    - Joe
categories:
    - "2020"
title: Introducing Lenovo Odometer
---
![Odometer](..\img/2020/odometer_image.png)

In some previous articles we have highlighted how you can collect Lenovo Updates History from a new WMI class in the root\Lenovo namespace. See <https://thinkdeploy.blogspot.com/2018/10/tracking-thininstaller-update-history.html>

We are now adding a new class under this namespace for the new "Odometer" feature found in the latest ThinkPads that were recently launched.  This feature keeps track of several metrics that can provide an indication of how a system has been used.  The metrics collected are:
<!-- more -->

* CPU Uptime - amount of time CPU has been active (in S0 state), counted in hours
* Shock events - based on detections from accelerometer where a delta reading of 1.75G(1.75m/s<sup>2</sup>) is detected
* Thermal events - registered high-temp conditions where CPU was throttled
* Battery cycles - number of charge cycles performed on battery
* SSD Read/Writes - number of block reads and writes on one or more internal SSDs

These counters are maintained by the Embedded Controller and the current values are exposed each time the system boots using SMBIOS Table data. In order to have this data stored in a meaningful way so it can be inventoried and collected by MEM Configuration Manager, we have created a PowerShell script (odometer.ps1, available from download link below) that can be implemented as either a scheduled task or used on demand to populate the Lenovo_Odometer class in WMI. 

Once you have run the PowerShell script on a system that supports Odometer you will be able to find the data in WMI as shown below:

![Odometer data class](..\img/2020/odometerdata.png)

As previously mentioned, you could also run the "odometer.ps1" using the very useful Run Scripts feature in the Config Manager console.  With that you will be able to get direct feedback from the device in the console as shown below:

![Run script](..\img/2020/runscript2.png)

The Config Manager hardware inventory can be extended to include the Lenovo_Odometer custom class using the "odometer.mof" file provide in the zip file linked below. In Config Manager you can import the file as a new Hardware Inventory Class in your Client Settings. For more information on extending hardware inventory, refer to the docs here:
<https://docs.microsoft.com/en-us/sccm/core/clients/manage/inventory/extend-hardware-inventory>

Once clients have reported inventory, you can create an SSRS report on the data that would like like the following:

![Report](..\img/2020/report2.png)

### Supported Systems

* ThinkPad P1 (Gen 3) / ThinkPad X1 Extreme (Gen 3)
* ThinkPad T14 (Intel/AMD)/ ThinkPad T14 Healthcare Edition / ThinkPad P14s
* ThinkPad T15 / ThinkPad P15s
* ThinkPad T14s (Intel/AMD)
* ThinkPad X13 (Intel/AMD)
* ThinkPad X13 Yoga
* ThinkPad X1 Carbon (8th gen)
* ThinkPad X1 Yoga (5th gen)
* ThinkPad P15v / ThinkPad T15p
* ThinkPad L14 (Intel)
* ThinkPad L15 (Intel)
* ThinkPad P15 / ThinkPad P15G
* ThinkPad P17

[Download .PS1 and .MOF files for this solution.](https://download.lenovo.com/cdrt/tools/odometer_01.zip)
