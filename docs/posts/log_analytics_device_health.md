---
date:
    created: 2022-09-27
    updated: 2026-10-27
authors:
    - Phil
categories:
    - "2022"
title: Reporting on Update Status, BIOS Level, CVEs, and More
---

![Azure Workbooks Icon](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/azure-workbooks.png)

This post will walk through deploying a [Remediation](https://learn.microsoft.com/mem/intune/fundamentals/remediations) script that will collect data from specific Lenovo WMI classes and send it to a [Log Analytics Workspace](https://learn.microsoft.com/azure/azure-monitor/logs/log-analytics-workspace-overview) in Azure Monitor using the [Logs Ingestion API](https://learn.microsoft.com/azure/azure-monitor/logs/logs-ingestion-api-overview).
<!-- more -->
!!! note
    This post was updated on **2026-10-22** to use the Logs Ingestion API. Support for the HTTP Data Collector API used by previous versions of this solution ended on **2026-09-14**. If you deployed a previous version, follow the steps in [Upgrading From the HTTP Data Collector API](#upgrading-from-the-http-data-collector-api).

We can then create an [Azure Monitor Workbook](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview) to report on the following data from your Lenovo devices.

!!! info ""
    This solution is not supported on ThinkBook

## Overview

> Summarizes the fleet in tiles: devices reporting, BIOS out of date, devices with pending or failed updates, batteries in red health, warranties expiring within 90 days, and devices that haven't reported in more than 7 days

![Overview](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/overview.jpg)

---

## Update Installation Status

> Summarizes installed, pending, failed, and not applicable updates for each device. Select a device to list its updates by title, version, severity, and install status.

![UpdateStatus](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image1.jpg)

![UpdateStatus](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image2.jpg)

---

## BIOS Level and CVEs

> Compares the installed BIOS on each device with the latest version in the Lenovo catalog and lists the CVEs addressed by the latest BIOS.

![BIOS](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image3.jpg)

![BIOS](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image4.jpg)

## Battery Health Status

> Lists the battery health of each ThinkPad. Select a slice of the chart to filter the list.

![BatteryHealth](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image5.jpg)

---

## Warranty End Date

> Groups devices by warranty status (expired, expiring within 90 days, expiring within 1 year, or more than 1 year) and lists the end date and days remaining for each device

![WarrantyEnd](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image6.jpg)

![WarrantyEnd](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image7.jpg)

## **Prerequisites**

Each report in the workbook relies on a data source on the endpoint.

| Report | WMI class | Requirement |
|---|---|---|
| Update Installation Status | **root\Lenovo\Lenovo_Updates** | Commercial Vantage or Lenovo Client Update |
| Battery Health Status | **root\Lenovo\Lenovo_Battery** | Commercial Vantage (ThinkPad only) |
| Warranty End Date | **root\Lenovo\Lenovo_WarrantyInformation** | Commercial Vantage |
| BIOS Level and CVEs | **root\CIMV2\Win32_BIOS** | None. The script installs the Lenovo Client Scripting Module to look up the available BIOS version and CVEs. |

### Commercial Vantage

[Lenovo Commercial Vantage](https://support.lenovo.com/solutions/hf003321-lenovo-vantage-for-enterprise) is required for the battery and warranty data, and can also provide the update installation status.

!!! info ""
    For a step by step guide on how to deploy Commercial Vantage through Intune, refer to this blog [article](https://blog.lenovocdrt.com/deploying-commercial-vantage-with-intune).

The following policies need to be enabled in order to create the WMI classes that will be inventoried

- Configure System Update
- Write warranty information to WMI table
- Write battery information to WMI table

If the battery or warranty policies aren't enabled, the status will return as "Unavailable" in the workbook.

### Lenovo Client Update

If you update devices with the [Lenovo Client Update](https://docs.lenovocdrt.com/guides/lcu/) module, it writes installation history to the same **Lenovo_Updates** WMI class when [Install-LnvUpdate](https://docs.lenovocdrt.com/guides/lcu/functions/install-lnvupdate/) is run with the `-ExportToWMI` parameter. The Update Installation Status report includes these results without Commercial Vantage installed.

### Required Files

- **Get-LenovoDeviceStatus.ps1**

- **LenovoDeviceHealth.json**

- **LenovoDeviceHealthDcr.json**

Download all three files from our GitHub [here](https://github.com/CDRT/Library/tree/master/lenovo-device-health/Log%20Analytics).

## **Azure Setup**

### Log Analytics Workspace

Assuming a Log Analytics Workspace has been created, note its **Name**, **Resource group**, and **Location** from the workspace **Overview** page. The Data Collection Endpoint and Data Collection Rule created in the next steps must be in the same region as the workspace.

### Create the Data Collection Endpoint

The Data Collection Endpoint (DCE) provides the URL the Remediation script sends inventory data to.

!!! note
    The **Microsoft.Insights** resource provider must be registered in the subscription before creating a DCE. Check its status under **Subscriptions** > **Resource providers**, or register it with `Register-AzResourceProvider -ProviderNamespace Microsoft.Insights`.

#### Azure Portal

- Login to the Azure [Portal](https://portal.azure.com)
- Go to **Monitor** > **Settings** > **Data Collection Endpoints**
- Click **Create**
- Enter a **Name**, such as **dce-lenovo-device-health**
- Select the **Subscription** and **Resource group**, and set the **Region** to the region of your workspace
- Click **Review + create**, then **Create**
- Once the deployment completes, click **Go to resource** and confirm the **Provisioning Status** is **Succeeded**
- On the **Overview** page, click **JSON View** and copy the following values:
    - **Resource ID**
    - **logsIngestion** > **endpoint** (for example, `https://dce-lenovo-device-health-a1b2.eastus-1.ingest.monitor.azure.com`)

#### PowerShell

Alternatively, create the DCE with the **Az.Monitor** module after signing in with `Connect-AzAccount`.

``` powershell
$dceParams = @{
    Name                           = 'dce-lenovo-device-health'
    ResourceGroupName              = 'rg-log-analytics'
    Location                       = 'eastus'
    NetworkAclsPublicNetworkAccess = 'Enabled'
}

$dce = New-AzDataCollectionEndpoint @dceParams

# Resource ID for the Data Collection Rule template
$dce.Id

# Logs ingestion endpoint for the Remediation script
$dce.LogIngestionEndpoint
```

Replace the **ResourceGroupName** and **Location** values with those of your workspace.

| Value | Used by |
|---|---|
| Resource ID | **dataCollectionEndpointResourceId** parameter of the Data Collection Rule template |
| Logs ingestion endpoint | **Get-LenovoDeviceStatus.ps1** |

### Create the App Registration

The Remediation script authenticates to the Logs Ingestion API with a Microsoft Entra app registration and client secret. The app doesn't need any API permissions. Its only access is the **Monitoring Metrics Publisher** role on the Data Collection Rule, which is assigned when the rule is deployed.

#### Microsoft Entra Admin Center

- Login to the Microsoft Entra [admin center](https://entra.microsoft.com)
- Go to **Entra ID** > **App registrations**
- Click **New registration**
- Enter a **Name**, such as **Lenovo Device Health Ingestion**
- Leave **Supported account types** set to **Single tenant only** and leave the **Redirect URI** blank
- Click **Register**
- On the **Overview** page, copy the **Application (client) ID** and **Directory (tenant) ID**
- Go to **Certificates & secrets** > **Client secrets** and click **New client secret**
- Enter a **Description**, select an **Expires** duration, and click **Add**
- Copy the secret **Value** right away. It's only displayed once.
- Go to **Entra ID** > **Enterprise apps**, search for the app name, and copy its **Object ID**

!!! note
    The Object ID on the **Enterprise apps** page belongs to the service principal and is the value the role assignment needs. The **Object ID** on the **App registrations** page is a different value.

#### PowerShell

Alternatively, create the app registration with the **Az.Resources** module after signing in with `Connect-AzAccount`.

``` powershell
$app = New-AzADApplication -DisplayName 'Lenovo Device Health Ingestion'
$servicePrincipal = New-AzADServicePrincipal -ApplicationId $app.AppId
$secret = New-AzADAppCredential -ApplicationId $app.AppId -EndDate (Get-Date).AddMonths(12)

# Values for the Data Collection Rule template and Remediation script
[PSCustomObject]@{
    TenantId                 = (Get-AzContext).Tenant.Id
    ClientId                 = $app.AppId
    ClientSecret             = $secret.SecretText
    ServicePrincipalObjectId = $servicePrincipal.Id
}
```

| Value | Used by |
|---|---|
| Directory (tenant) ID | **Get-LenovoDeviceStatus.ps1** |
| Application (client) ID | **Get-LenovoDeviceStatus.ps1** |
| Client secret value | **Get-LenovoDeviceStatus.ps1** |
| Enterprise app Object ID | **servicePrincipalObjectId** parameter of the Data Collection Rule template |

!!! warning
    The client secret is stored in plain text in the Remediation script on each device. The secret can only send data to this Data Collection Rule and can't read any data from the workspace. Ingestion stops when the secret expires, so set a reminder to create a new secret and update the Remediation script before the expiration date.

### Deploy the Data Collection Rule

The **LenovoDeviceHealthDcr.json** template deploys the following resources:

- The **Lenovo_Device_Health_CL** and **Lenovo_Device_Properties_CL** custom tables in your workspace
- A Data Collection Rule (DCR) that receives data through the DCE and writes it to both tables
- The **Monitoring Metrics Publisher** role assignment on the DCR for the app registration

The template uses the same column names as the original HTTP Data Collector API tables, so the workbook queries work with both new and migrated tables.

#### Upgrading From the HTTP Data Collector API

If you already have the **Lenovo_Device_Health_CL** and **Lenovo_Device_Properties_CL** tables from a previous version of this solution, migrate them to Data Collection Rule-based tables before deploying the template. Existing data is retained.

``` powershell
$migrateParams = @{
    ResourceGroupName = 'rg-log-analytics'
    WorkspaceName     = 'law-lenovo'
}

Invoke-AzOperationalInsightsMigrateTable @migrateParams -TableName 'Lenovo_Device_Health_CL'
Invoke-AzOperationalInsightsMigrateTable @migrateParams -TableName 'Lenovo_Device_Properties_CL'
```

Confirm the migration in the Azure portal before deploying the template:

- Go to your **Log Analytics workspace** > **Settings** > **Tables**
- Search for **Lenovo_Device**
- Verify the **Type** column for both tables shows **Custom table**. A table that hasn't been migrated shows **Custom table (classic)**.

When deploying the template, set **deployTables** to **false** so the migrated tables are left as they are. Deploying the template against tables that haven't been migrated fails with **Data collection rule is invalid (Code: InvalidPayload)**.

!!! warning
    Table migration can't be rolled back. Devices running the previous version of **Get-LenovoDeviceStatus.ps1** continue to send data to migrated tables until the updated Remediation script is deployed.

#### Azure Portal

- Login to the Azure [Portal](https://portal.azure.com)
- Search for **Deploy a custom template** and select it
- Click **Build your own template in the editor**
- Click **Load file**, browse to **LenovoDeviceHealthDcr.json**, and click **Save**
- Select the **Subscription** and the **Resource group** that contains your workspace
- Fill in the parameters

| Parameter | Value |
|---|---|
| Workspace Name | Name of your Log Analytics workspace |
| Location | Region of your workspace. Defaults to the region of the resource group. |
| Data Collection Endpoint Resource Id | Resource ID of the DCE |
| Data Collection Rule Name | Defaults to **dcr-lenovo-device-health** |
| Service Principal Object Id | Object ID from **Enterprise apps** |
| Deploy Tables | **true** for new deployments, **false** for migrated tables |

- Click **Review + create**, then **Create**
- Once the deployment completes, select **Outputs** and copy the **dcrImmutableId** value

#### PowerShell

Alternatively, deploy the template with the **Az.Resources** module.

``` powershell
$deploymentParams = @{
    ResourceGroupName       = 'rg-log-analytics'
    TemplateFile            = '.\LenovoDeviceHealthDcr.json'
    TemplateParameterObject = @{
        workspaceName                    = 'law-lenovo'
        dataCollectionEndpointResourceId = '/subscriptions/<subscription-id>/resourceGroups/rg-log-analytics/providers/Microsoft.Insights/dataCollectionEndpoints/dce-lenovo-device-health'
        servicePrincipalObjectId         = '<enterprise-app-object-id>'
        deployTables                     = $true
    }
}

$deployment = New-AzResourceGroupDeployment @deploymentParams

# DCR immutable ID for the Remediation script
$deployment.Outputs.dcrImmutableId.Value
```

| Value | Used by |
|---|---|
| dcrImmutableId | **Get-LenovoDeviceStatus.ps1** |

!!! note
    If you leave **servicePrincipalObjectId** blank, assign the role manually. Open the DCR, go to **Access control (IAM)** > **Add role assignment**, select **Monitoring Metrics Publisher**, and add the app registration as a member.

    Data can take 10-15 minutes to appear in new tables after the first upload.

## Deploying the Remediation

Once you have Commercial Vantage installed and the necessary policies enabled on your endpoints, it's time to deploy the Remediation script to collect inventory. The script will install the [Lenovo Client Scripting Module](https://www.powershellgallery.com/packages/Lenovo.Client.Scripting) from the PowerShell Gallery on the endpoint to help gather certain data, such as Available BIOS versions and CVEs.

Replace the following variables in the **Get-LenovoDeviceStatus.ps1** script with the values collected in the previous steps.

| Variable | Value |
|---|---|
| **$TenantId** | Directory (tenant) ID of the app registration |
| **$ClientId** | Application (client) ID of the app registration |
| **$ClientSecret** | Client secret value of the app registration |
| **$DceEndpoint** | Logs ingestion endpoint of the Data Collection Endpoint |
| **$DcrImmutableId** | **dcrImmutableId** output of the Data Collection Rule deployment |

- Login to the Microsoft Endpoint Manager [admin center](https://endpoint.microsoft.com)
- Navigate to **Devices** > **Scripts and remediations**
- Click **Create**
- Enter a name for the script and click **Next**
- Browse to **Get-LenovoDeviceStatus.ps1** for the **Detection script file**
- Select **Yes** to **Run script in 64-bit PowerShell** and click **Next**
- Assign to a group and edit the schedule. Set **Frequency** to **Daily**, **Repeats every** to **7 days**, and choose a **Start time**.
- **Create**

!!! note
    The workbook reports on the last 30 days of data by default and shows the most recent inventory of each device. A weekly schedule gives each device several chances to report within that window, including devices that are offline for a run. Running the Remediation more often adds rows to the tables without affecting the workbook counts.

As devices receive the policy, review the **Device status** blade to verify if devices don't have Commercial Vantage installed or the necessary policies enabled. You can review this by adding the **Pre-remediation detection output** column.

## Reporting

Once devices run the Remediation, confirm the data is arriving in your workspace.

- Login to the Azure [Portal](https://portal.azure.com)
- Go to your **Log Analytics workspace** > **Logs**
- If the query editor opens in **Simple mode**, switch to **KQL mode** with the toggle in the top right
- Run the following queries to view the most recent data in each table

``` kusto title="Device properties"
Lenovo_Device_Properties_CL
| where TimeGenerated > ago(1d)
| project TimeGenerated, Hostname_s, MTM_s, Product_s, BatteryHealth_s, WarrantyEnd_s,
          InstalledBiosVersion_s, AvailableBiosVersion_s, CVE_s
```

``` kusto title="Update status"
Lenovo_Device_Health_CL
| where TimeGenerated > ago(1d)
| project TimeGenerated, Hostname_s, PackageID_s, Title_s, Version_s, Severity_s, Status_s
```

Both tables are also listed under **Custom Logs** in the table list on the left side of the query editor.

To review the columns of each table, go to **Settings** > **Tables**, click the **...** menu on the table, and select **Edit schema**.

Everything being inventoried here can be found on the device in the **root/Lenovo** WMI namespace under one of the following classes

- **Lenovo_Updates**
- **Lenovo_Battery**
- **Lenovo_WarrantyInformation**

### Adding the Workbook

Add the sample Workbook to your Log Analytics Workspace

- Login to the Azure [Portal](https://portal.azure.com)
- Go to **Log Analytics Workspace** > **Workbooks**
- Click **+New**
- Click the **Advanced Editor** button

![LA-Workspace](https://cdrt.github.io/mk_blog/img/2022/log_analytics_device_health/image6.jpg)

- Clear the contents and copy/paste the contents from **LenovoDeviceHealth.json** here
- Click **Apply**

Use the parameters at the top of the workbook to filter every tab.

| Parameter | Description |
|---|---|
| **Time Range** | How far back to look for inventory. Defaults to 30 days. Ranges longer than the workspace retention period return no additional data. |
| **Product** | Limits the reports to one or more product families, such as **ThinkPad X1 Carbon Gen 14** |

To export a list of devices, filter the tab and click **Export to Excel** above the grid. The export contains the rows that match the selected filters. For example, set **BIOS Status** to **Out of Date** on the **BIOS** tab to export the devices that need a BIOS update.

Happy reporting!
