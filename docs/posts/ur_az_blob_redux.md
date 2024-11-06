---
date:
    created: 2022-01-05
    updated: 2024-11-05
authors:
    - Phil
categories:
    - "2022"
title: Revisiting Update Retriever and Azure Blob Storage
---

This is a follow up to a previous [article](https://blog.lenovocdrt.com/hosting-a-repository-in-an-azure-blob) that walked through hosting an on-prem Update Retriever repository in an Azure Blob Storage.

If you've started leveraging cloud storage for your driver repository, this solution may be of interest to you if you're at a roadblock with how to deploy BIOS updates to your endpoints.

The main focus of this article is a rather hot topic: **Silently installing BIOS and firmware updates**.

<!-- more -->

Historically, BIOS packages prompted the user to proceed with the update followed by a forced reboot. In later models, BIOS and firmware packages present a 5 minute countdown timer before the system force reboots. In an enterprise, suppressing these prompts and preventing any forced reboots is extremely important.

This solution assumes you have an on-prem Update Retriever repository and [Azure Blob Storage](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-introduction) already in place.

## Entra App Registration

For authentication, I'm using a [Service Principal](https://docs.microsoft.com/azure/active-directory/develop/howto-create-service-principal-portal#register-an-application-with-azure-ad-and-create-a-service-principal), which the example script uses.

Generate a **client secret** and save its value. It will be used in the script, along with the application (client) ID.

## Service Principal Role Assignments

After you've registered a new Entra app, you'll need to assign a few roles to the service principal to allow content to be transferred to the Blob container. In my tenant, I set the scope at the resource level.

1. Browse to your Storage Account where the Blob container resides and select **Access control (IAM)**.

2. Select **Add**, then select **Add role assignment**.

3. In the **Role** tab, select **Storage Account Contributor** and **Storage Blob Data Contributor**.

4. Select **Next**.

5. On the **Members** tab, for **Assign access to**, select **User, group, or service principal**.

6. Select **Select members**. Search for the Entra app you created earlier since Entra apps aren't displayed in the available options by default.

7. Select the **Select** button, then select **Review + assign**.

## Downloading BIOS and Firmware Updates

!!! info ""
    Since we have to modify the packages, make sure your Update Retriever repository is configured as a **Local** repository.

Launch Update Retriever and select the models you want to get updates for.

Select BIOS and Firmware packages to download (and/or drivers).

Once the packages are downloaded, click **Manage repository** > **Update view**, you'll see the **Reboot type** is **Reboot Delayed**.

![Update Retriever](https://cdrt.github.io/mk_blog/img/2022/ur_az_blob_redux/image1.jpg)

## XML Change and Repository Sync

With some PowerShell, we can save time by automatically altering the package XML descriptors to support a completely silent installation of BIOS and firmware updates using Thin Installer. After running the code, the output will show which package XML's were modified or skipped, the AzCopy statistics, and the log file path for more details.

![AzCopy](https://cdrt.github.io/mk_blog/img/2022/ur_az_blob_redux/image2.jpg)

If you refresh the view in Update Retriever, you'll see the **Reboot type** is now **Requires a reboot**.

![Update Retriever](https://cdrt.github.io/mk_blog/img/2022/ur_az_blob_redux/image3.jpg)

The script can be found on my GitHub:

<https://github.com/philjorgensen/Azure/blob/main/Blob/Sync-Repositories.ps1>

## Scenario

The following Thin Installer command line will pull these packages down for a silent installation

```cmd
.\ThinInstaller.exe /CM -search A -action INSTALL -includerebootpackages 3 -packagetypes 3,4 -noicon -repository https://storageaccount.blob.core.windows.net/bios-repository -ignorexmlsignature -noreboot -exporttowmi
```

!!! info ""
    **-includerebootpackages 3** filters for Reboot required packages
    
    **-packagetypes 3,4** filters for only BIOS and Firmware packages

## Reference

Refer to the System Update Deployment Guide for additional Thin Installer usage

<https://docs.lenovocdrt.com/guides/sus/su_dg/su_dg_ch5/#52-thin-installer>
