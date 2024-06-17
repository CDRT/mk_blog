---
date:
    created: 2022-03-15
authors:
    - Phil
categories:
    - "2022"
title: Installing Updates from the Lenovo 3rd Party Catalog in a ConfigMgr Operating System Deployment
---

In some scenarios, there may be the desire to leverage the Lenovo Updates Catalog to apply applicable updates during an OS deployment task sequence. This article will cover how this might be achieved.
<!-- more -->
## Prerequisites

- Enable third-party updates on your Software Update Point - **<https://docs.microsoft.com/mem/configmgr/sum/deploy-use/third-party-software-updates>**

- Subscribe to the [Lenovo Third-Party Update Catalog](https://blog.lenovocdrt.com/#/2020/lucv3) in ConfigMgr - **<https://docs.microsoft.com/en-us/mem/configmgr/sum/deploy-use/third-party-software-updates#subscribe-to-a-third-party-catalog-and-sync-updates>**

---

After subscribing and synchronizing the Lenovo Updates Catalog, open the Software Update Point Component Properties and check the box next to the **Lenovo** product tree to sync Lenovo Updates and the LUCAgent.

![Selecting products](\img/2022/configmgr_osd_updates/image1.jpg)

Initiate a site-wide synchronization of software updates to sync the metadata for all updates in the Lenovo catalog. Once the sync has completed, go to the **Software Library** workspace, expand **Software Updates**, and select the **All Software Updates** node. All Lenovo updates should be populated here.

In this example, I'm going to publish the **Lenovo Updates Catalog Agent**. This is a light weight tool that performs data conversions in WMI to assist with the evaluation of the installation and applicability rules for each update in the Lenovo Updates Catalog. The installation and applicability rules determine if an update is currently installed or if an update is applicable to the device. This is **required** on all Lenovo products if you plan on publishing and deploying other updates.

In the **All Software Updates** node, filter the updates by adding the following criteria:

- Title contains **Lenovo Updates Catalog Agent**
- Expired set to **No**
- Superseded set to **No**
- Vender equals **Lenovo**

!!! info ""
    Optionally, save this search by clicking **Save Current Search** in the ribbon bar

![Search criteria](\img/2022/configmgr_osd_updates/image2.jpg)

Select the Lenovo Updates Catalog Agent update and select **Publish Third-Party Software Update Content** to download the update binaries, which is stored in the WSUSContent directory. Initiate a site-wide synchronization of software updates. Once complete, the icon beside the update should flip from blue (metadata-only) to green (deployable).

I will create an Automatic Deployment Rule to add the LUCAgent (and any future versions) to a Software Update Group, which can then be deployed to a Device Collection.

In the **Software Library** workspace, expand **Sofware Updates** and select **Automatic Deployment Rules**.

- Select **Create Automatic Deployment Rule** from the ribbon bar.
- Enter **Lenovo Updates Catalog Agent** for the name of the rule.
- Specify the **All Unknown Computers** collection.
- Select the radio button **Add to an existing Software Update Group.
- Tick the box to **Enable the deployment after this rule is run**.

On the **Deployment Settings** page, set the type of deployment to **Required**.
Set the following property filters and search criteria as follows:

- Superseded **No**
- Title **Lenovo Updates Catalog Agent**
- Vendor **Lenovo**

Click Preview and 1 updates should return

![Preview](\img/2022/configmgr_osd_updates/image3.jpg)

Specify the recurring schedule for the rule. I chose to run the rule after any software update point synchronization.

Further in the wizard, select the option to **Create a new deployment package**. This will contain the update(s) associated with the ADR.

- Enter the name **Lenovo Updates Catalog Agent**
- Specify the source path to where the deployment package will reside.
- Add the Distribution Points or distribution point groups to host the content.
- Specify the download location for the ADR. My site server has an Internet connection so I'll choose to **Download software updates from the Internet**.

Proceed through the wizard to complete the creation of the ADR, which will then run.

Select the **Software Update Groups** node to verify the Lenovo Updates Catalog Agent SUG has been created and populated with the update.

![Software Updates Groups](\img/2022/configmgr_osd_updates/image4.jpg)

---

## WSUS Certificate and Installation Script

To successfully install the LUCAgent (or any Third-Party update) in an operating system deployment, the third-party Wsus signing certificate needs to be installed in the **Trusted Root** and **Trusted Publishers** certificate stores. The Windows Update agent policy to **Allow signed updates for an intranet Microsoft update service location** will also need to be set. This is handled through Client Settings under the **Software Updates** tab.

However, in an operating system deployment, the device can't receive this client policy so these tasks have to be handled beforehand.

The below PowerShell commands will be used to accomplish this

!!! info ""
    Update the script to match the name of the Wsus signing certificate when you exported it.

```powershell
# Set registry entry
New-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate' -Name AcceptTrustedPublisherCerts -PropertyType DWord -Value 1
Write-Output "Registry Entry set"

# Import certificates into Root and Trusted Publisher store
certutil.exe -addstore -f "TrustedPublisher" .\Import-WsusSigningCertificate.ps1
certutil.exe -addstore -f "Root" .\Import-WsusSigningCertificate.ps1
Write-Output "Wsus Signing Certificate imported"
```

Save this script as **Import-WsusSigningCertificate.ps1** to a source location. Export the Wsus signing certificate from a workstation or your site server and save it to the same location.

In my lab, I saved both items to **\\share\ConfigMgr\Content\OSD\Certificates**

![Source files](\img/2022/configmgr_osd_updates/image5.jpg)

---

## Operating System Deployment Task Sequence

In the Configuration Manager console, select the **Software Library** workspace, expand **Application Management**, and select the **Packages** node.

- Select **Create Package**
- Enter a name for the package.
- Tick the box beside **This package contains source files**
  - Set the source folder to the location where **Import-WsusSigningCertificate.ps1** and the Wsus signing certificate reside.
- Do not create a program for the package.
- Distribute the package to your Distribution Points and/or Distribution Point Groups.

Create a new or Task Sequence or edit an existing one. After the **Setup Windows and Configuration Step**, add a **Run PowerShell Script** step.

- Name the step **Import Certificate**
- Select the package that you just created and enter the script name **Import-WsusSigningCertificate.ps1**
- Set the PowerShell execution policy to **Bypass**

![Run PowerShell Script](\img/2022/configmgr_osd_updates/image6.jpg)

Add an **Install Software Updates** step

- Select **Required for installation - Mandatory software updates only**

Deploy this Task Sequence to an Unknown Lenovo device. Monitoring the **smsts.log**, I see the Windows Update agent policy to accept trusted publishers has been set and the certificate has been added to the certificate stores.

![Log](\img/2022/configmgr_osd_updates/image7.jpg)

A bit further down, The installation of the LUC Agent has completed successfully.

![Success](\img/2022/configmgr_osd_updates/image8.jpg)

If these steps weren't added, you would see a **0x800b0109** error in the log, which translates to "A certificate chain processed, but terminated in a root certificate which is not trusted by the trust provider."
