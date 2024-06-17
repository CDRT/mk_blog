---
date:
    created: 2018-10-22
authors:
    - Phil
categories:
    - "2018"
title: Tracking Thin Installer Update History With ConfigMgr Current Branch
---

Due to customer feedback, Lenovo is introducing a couple of new features focused on tracking updates.  Traditionally, updates installed on the client were logged in the **Updates_log(timestamp).txt**.

Admittedly, parsing through the log to find out which updates were skipped, installed, or failed is not that easy. If you're deploying a task sequence to your Lenovo systems that runs Thin Installer, you can only see if Thin Installer runs or not. How do you tell which updates installed without logging into each system and checking the logs?

A new switch can now be added to your Thin Installer command line that will do the following upon execution:
<!-- more -->
- Create a new **Lenovo** WMI namespace and **Lenovo_Updates** class
- Adds the following Properties to the Class
  - **PackageID** - Listed as Update ID in Update Retriever
  - **Title** - Friendly name of the update
  - **Status** - Possible values can be:
    - NotApplicable
    - AlreadyInstalled
    - Applicable
    - NotInstalled
    - DownloadFailed
    - InstallFailed
    - InstallSuccess
  - **AdditionalInfo** - Provides more detail if the Status equaled Download Failed.  Possible values can be:
    - InstallerFileNotExist
    - InstallerFileCrcNotMatch
    - ExternalFileNotExist
    - ExternalFileCrcNotMatch
  - **Version** - The version associated with the update installed

A sample Thin Installer command line using the new **-exporttowmi** switch

``` cmd
ThinInstaller.exe /CM -search A -action INSTALL -repository \\URrepo\LenovoUpdates -noicon -includerebootpackages 3 -noreboot -exporttowmi -log "%SystemDrive%\Program Files (x86)\ThinInstaller\Logs"
```

## Exploring WMI

Launching WMIExplorer on a target system and navigating to the newly created Namespace/Class, we can see the 5 properties have been created

![WMI Explorer - Lenovo Namespace](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image1.jpg)

On the Instances tab, you'll notice 17 entries. This means that 17 applicable updates to this system were found in my Update Retriever repository.

![WMI Explorer - Lenovo Updates](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image2.jpg)

Each instance will display the PackageID of the update it found. If you click on an instance, you can see the result of the update that attempted to install on the system. In this case, the latest Chipset Software is already installed.

![WMI Explorer - Package Status](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image3.jpg)

Reviewing another instance, a newer version of the Camera Driver was installed successfully

![WMI Explorer - Package Status](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image4.jpg)

One more example, this Bluetooth Driver returned as Not Applicable because it only applies to Windows 10 1703. The system I ran Thin Installer on is Windows 10 1803.

![WMI Explorer - Package Status](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image5.jpg)

## Collecting the data with Hardware Inventory

Now that the data is in WMI, it can be collected. Copy the contents below into NotePad and save it as a .MOF file, which you will then import as a new Hardware Inventory Class in your Client Settings.  For more information on extending hardware inventory, refer to the MS docs [here](https://docs.microsoft.com/sccm/core/clients/manage/inventory/extend-hardware-inventory).

``` txt
[SMS_Report, SMS_Group_Name("Lenovo_Updates"), SMS_Class_Id("Lenovo_Updates"),Namespace ("root\\\\Lenovo")]
class Lenovo_Updates: SMS_Class_Template
{
 [key, SMS_Report] string PackageID;
 [SMS_Report] string Title;
 [SMS_Report] string Status;
 [SMS_Report] string AdditionalInfo;
 [SMS_Report] string Version;
};
```

![WMI Explorer - Package Status](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image6.jpg)

Once the client has received the updated Client Settings and the Hardware Inventory cycle has processed, you can start up Resource Explorer and review the data.

![WMI Explorer - Package Status](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image7.jpg)

## Tracking Thin Installer Version and Last Scan

We can take this a step further and track the version of Thin Installer on the system and when a scan was last performed to install updates. Thin Installer adds a few registry keys and stamps the Version and Last Scan Date every time it executes.

Since these are registry keys that need to be collected, you'll need to append the **configuration.mof** specifying the data to be consumed as part of the Hardware Inventory cycle.

[CTGlobal](http://blog.ctglobalservices.com/configuration-manager-sccm/kea/how-to-get-registry-information-into-hardware-inventory/) and [Enhansoft](https://www.enhansoft.com/blog/how-to-use-regkeytomof) walk through how to do this using the RegKeytoMof tool.

Here's the information to be copied into the configuration.mof

``` txt
#pragma namespace ("\\\\.\\root\\cimv2")
#pragma deleteclass("ThinInstaller", NOFAIL)
[DYNPROPS]
Class ThinInstaller
{
[key] string KeyName;
String Version;
String LastScanDate;
};

[DYNPROPS]
Instance of ThinInstaller
{
KeyName="ThinInstaller";
[PropertyContext("Local|HKEY_LOCAL_MACHINE\\SOFTWARE\\WOW6432Node\\Lenovo\\ThinInstaller|Version"),Dynamic,Provider("RegPropProv")] Version;
[PropertyContext("Local|HKEY_LOCAL_MACHINE\\SOFTWARE\\WOW6432Node\\Lenovo\\ThinInstaller|LastScanDate"),Dynamic,Provider("RegPropProv")] LastScanDate;
};
```

Upon the next Hardware Inventory cycle, these registry keys will be collected and can be reviewed in Resource Explorer

![WMI Explorer - Package Status](https://cdrt.github.io/mk_blog/img/2018/ti_update_history_configmgr/image8.jpg)
