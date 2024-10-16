---
date:
    created: 2024-08-19
authors:
    - Joe
categories:
    - "2024"
title: "Introducing: SU Helper Utility"
---

![SUHelper](https://cdrt.github.io/mk_blog/img/2024/intro_suhelper/app_logo.png)

The SU Helper Utility provides a command-line interface for triggering the System Update process of Commercial Vantage. This seperate utility is not included with Commercial Vantage and must also be deployed to clients where it is needed. Let's see some of the things we can do with this utility.
<!-- more -->
You can view the reference for the tool here: [SU Helper Reference](https://docs.lenovocdrt.com/guides/cv/suhelper/)

## Installing SU Helper

## Use Cases for SU Helper

!!! note
    When using SU Helper, you must always specify the "-autoupdate" parameter first. The only exception to this is when using "-help" to get a reference to the possible parameters at the command line.

### Trigger an Autoupdate Session

The simplest function of SU Helper would be to just trigger an Autoupdate session to run on demand. Since Commercial Vantage is architected as a UWP application instead of a standard Win32 applciation, this as previously been difficult to do. Now with SU Helper you can simply run:

```suhelper.exe -autoupdate```

### Filtering Updates

Let's consider a case where you may want to trigger an Autoupdate session to only update the BIOS if needed. With SU Helper you can simply run:

```suhelper.exe -autoupdate -packagetype 3```

Package type numbers must be from the following list:

    {0} : Others
    {1} : Application
    {2} : Driver
    {3} : Bios
    {4} : Firmware

### Install a Specific Update

There may be a case where a particular update needs to be installed immediately, outside of the normal schedule for updates. With SU Helper you can specify the Package ID for one or more updates that are required to be installed and only these updates will be applied during the Autoupdate session initiated.

```suhelper.exe -autoupdate -include n47uj02w```

When the Autoupdate session is triggered, the catalog of available updates is searched for the specified Package ID. If that package is not in the catalog, or if the package has already been installed, the Autoupdate session will simply end. Only if the update is applicable and listed in the catalog will it be installed.

You can easily find the package IDs for updates by either using Update Retriever or the [Driver & Software Matrix for IT Admins](https://download.lenovo.com/cdrt/tools/drivermatrix/dm_2.html). The latter has been updated to provide checkboxes in the list of search results so that one or more updates can be selected, then clicking the **Copy Package ID(s)** button will copy a string to the clipboard consisting of the selected Package IDs separated by a comma. This string can be used after the -include parameter in the command line.

### Exclude a Specific Update From Being Installed

In some cases, you may encounter an update that is not compatible with something else in your environment and you may want your devices running Commercial Vantage to ignore this particular update. This can be accomplished by running:

```suhelper.exe -autoupdate -exclude n47uj02w```

This will add the specified Package ID to a list of excluded packages on the device. When any subsequent Autoupdate sessions or even manual Check for Updates sessions are run, the updates listed in the exclude list will be ignored. If the update becomes required again, it can be removed from the excluded list and installed by using the "-include" parameter with the Package ID.

## Closing

To get the full details of all the parameters and their usage, please be sure to check out the [SU Helper Reference guide](https://docs.lenovocdrt.com/guides/cv/suhelper/).
