---
date:
    created: 2022-09-27
    updated: 2024-02-23
authors:
    - Phil
categories:
    - "2022"
title: Reporting on Update Status, BIOS Level, CVEs, and More
---

![Azure Workbooks Icon](..\img/2022/log_analytics_device_health/azure-workbooks.png)

This post will walk through deploying a [Remediation](https://learn.microsoft.com/mem/intune/fundamentals/remediations) script that will collect data from specific Lenovo WMI classes using the [HTTP Data Collector API](https://learn.microsoft.com/azure/azure-monitor/logs/data-collector-api). As a result, the data will be sent to a [Log Analytics Workspace](https://learn.microsoft.com/azure/azure-monitor/logs/log-analytics-workspace-overview) in Azure Monitor.
<!-- more -->
!!! warning
    The HTTP Data Collector API will be deprecated on **2026/09/14**

We can then create an [Azure Monitor Workbook](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview) to report on the following data from your Lenovo devices.

!!! info ""
    This solution is not supported on ThinkBook

## Update Installation Status

> Lists all updates by package ID and title with their respective install statuses

![UpdateStatus](..\img/2022/log_analytics_device_health/image1.jpg)

![UpdateStatus](..\img/2022/log_analytics_device_health/image2_1.jpg)

![UpdateStatus](..\img/2022/log_analytics_device_health/image2_2.jpg)

---

## BIOS Level and CVEs

> Displays BIOS level information for each model and if the model is affected by any CVEs.

![BIOS](..\img/2022/log_analytics_device_health/image3_1.jpg)

![BIOS](..\img/2022/log_analytics_device_health/image3_2.jpg)

## Battery Health Status

> Lists all endpoints and health of their battery.

![BatteryHealth](..\img/2022/log_analytics_device_health/image2.jpg)

---

## Warranty End Date

> Lists the warranty end dates of all endpoints

![WarrantyEnd](..\img/2022/log_analytics_device_health/image3.jpg)

## **Prerequisites**

The data gathered relies on [Lenovo Commercial Vantage](https://support.lenovo.com/solutions/hf003321-lenovo-vantage-for-enterprise) being installed on your endpoints.

!!! info ""
    For a step by step guide on how to deploy Commercial Vantage through Intune, refer to this blog [article](https://blog.lenovocdrt.com/#/2020/cv_intune_deploy).

The following policies need to be enabled in order to create the WMI namespaces that will be inventoried, along with their respective classes

- Configure System Update
- Write warranty information to WMI table
- Write battery information to WMI table

If these polices aren't enabled, the status will return as "Unavailable" in the workbook.

### Required Files

- **Get-LenovoDeviceStatus.ps1**

- **LenovoDeviceHealth.json**

Download both files from our GitHub [here](https://github.com/CDRT/Library/tree/master/lenovo-device-health/Log%20Analytics).

## **Log Analytics Workspace**

Assuming a Log Analytics Workspace has been created, you'll need the **Workspace ID** and **Primary Key**. This can be found under **Agents management** of your workspace.

![LA-Workspace](..\img/2022/log_analytics_device_health/image5.jpg)

## Deploying the Remediation

Once you have Commercial Vantage installed and the necessary policies enabled on your endpoints, it's time to deploy the Remediation script to collect inventory. The script will install the [Lenovo Device Management Module](https://blog.lenovocdrt.com/#/2024/intro_ldmm) on the endpoint to help gather certain data, such as Available BIOS versions and CVEs.

!!! info ""
    Replace the **$customerID** and **$sharedKey** variables in the **Get-LenovoDeviceStatus.ps1** script

- Login to the Microsoft Endpoint Manager [admin center](https://endpoint.microsoft.com)
- Navigate to **Devices** > **Scripts and remediations**
- Click **Create**
- Enter a name for the script and click **Next**
- Browse to **Get-LenovoDeviceStatus.ps1** for the **Detection script file**
- Select **Yes** to **Run script in 64-bit PowerShell** and click **Next**
- Assign to a group and set the schedule for the script package to run.
- **Create**

As devices receive the policy, review the **Device status** blade to verify if devices don't have Commercial Vantage installed or the necessary policies enabled. You can review this by adding the **Pre-remediation detection output** column.

## Reporting

You can check what data is being collected by looking at the **Custom fields** tab in **Custom logs** in your Workspace

![CustomFields](..\img/2022/log_analytics_device_health/image4.jpg)

Everything being inventoried here can be found on the device in the **root/Lenovo** WMI namespace under one of the following classes

- **Lenovo_Updates**
- **Lenovo_Battery**
- **Lenovo_Warranty**

### Adding the Workbook

Add the sample Workbook to your Log Analytics Workspace

- Login to the Azure [Portal](https://portal.azure.com)
- Go to **Log Analytics Workspace** > **Workbooks**
- Click **+New**
- Click the **Advanced Editor** button

![LA-Workspace](..\img/2022/log_analytics_device_health/image6.jpg)

- Clear the contents and copy/paste the contents from **Lenovo-DeviceStatus.json** here
- Click **Apply**

Happy reporting!
