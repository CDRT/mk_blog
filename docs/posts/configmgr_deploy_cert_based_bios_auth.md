---
date: 2025-09-03
authors:
    - Phil
categories:
    - "2025"
title: Deploying Certificate-based BIOS Authentication with Microsoft Configuration Manager
---

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image1.png)

This guide will demonstrate how to convert the BIOS security of a Lenovo Think product, protected by a Supervisor Password, to a digital certificate-based authentication mechanism using a Configuration Manager task sequence.
<!-- more -->

## Overview

For those wanting to harden BIOS access even further by switching to certificate-based authentication, let's explore how to accomplish this with the power of a task sequence.

### Prerequisites

Before proceeding, this solution assumes the following:

- Your devices are currently secured with a Supervisor Password
- Code signing certificate file (.pem) with public/private keys has been generated

!!! info
    Follow this blog [article](https://blog.lenovocdrt.com/certificate-based-bios-authentication/#getting-started) for a more in-depth guide on how to generate a code signing certificate.

## Create the Packages

In my lab, I've created 2 legacy Packages. One for my **.pem** certificate file that will be installed on my test device and another containing the **LnvBiosCerts** PowerShell module that will be copied to the PowerShell modules folder.

In the console, navigate to **Software Library** > **Application Management** > **Packages** and create 2 new Packages:

- **Lenovo BIOS Certificate** - Package source path contains your **.pem** certificate file

- **PSModule - LnvBiosCerts** - Package source path contains the extracted contents of the [LnvBiosCerts module](https://docs.lenovocdrt.com/guides/lbct/)
  - Example source path directory structure should look like:
      - \\\share\ConfigMgr\Content\Software\Lenovo\Modules\LnvBiosCertTool_1.0.2
          - LnvBiosCerts
              - 1.0.2
                  - LnvBiosCerts.psd1
                  - LnvBiosCerts.psm1
          - LnvBiosCertInterface.ps1 (optional)

Distribute packages to your distribution points

## Create the Task Sequence

This example is short and not overly complex, however, a few of the steps consist of critical WQL queries that checks for the appropriate BIOS Password State value required to convert to certificate-based auth.

The layout is below. Let's walk through the steps

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image2.png)

- **Set Task Sequence Variable**
    - Name: **isThink**
    - Value: **true**
    - Condition: **WMI Query**
        - WMI Namespace: **root\cimv2**

```dos
 SELECT * FROM Win32_ComputerSystemProduct
 WHERE Version LIKE 'ThinkPad%'
   OR Version LIKE 'ThinkCentre%'
   OR Version LIKE 'ThinkStation%'
```

This step will only set the variable if the device is a ThinkPad, ThinkCentre, or ThinkStation


![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image3.png)

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image4.png)

- **Set Task Sequence Variable**
    - Name: **isSvpSet**
    - Value: **true**
    - Condition: **WMI Query**
        - WMI Namespace: **root\WMI**

```dos
SELECT PasswordState FROM Lenovo_BiosPasswordSettings WHERE
(PasswordState = 2) OR
(PasswordState = 3) OR
(PasswordState = 6) OR
(PasswordState = 7) OR
(PasswordState = 66)
```

This step will only set the variable if the value of the BIOS Password State matches. Reference this [table](https://docs.lenovocdrt.com/ref/bios/wmi/wmi_guide/?h=passwordstate#detecting-password-state) for what each value translates to.


![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image5.png)

- **Copy LnvBiosCerts Module Group**
    - Condition: Task Sequence Variable **isThink equals "true"**

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image6.png)

- **Run Command Line** - This command uses xcopy to copy the LnvBiosCerts module to the WindowsPowerShell\Modules directory
    - Name: **Copy to PS5 Modules**
    - **Disable 64-bit file system redirection**
    - Tick **Package** and choose the LnvBiosCerts Package created earlier

```dos
xcopy ".\*.*" "%ProgramFiles%\WindowsPowerShell\Modules" /e /i /y
```


![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image7.png)

- **Convert to Cert Auth Group**
    - Condition: Task Sequence Variable **isSvpSet equals "true"**

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image8.png)

- **Run Command Line**
    - Name: **Import LnvBiosCerts Module**

```dos
powershell.exe -ExecutionPolicy Bypass -Command "Import-Module LnvBiosCerts -Force"
```

- **Set Task Sequence Variable**
    - Name: [**OSDDoNotLogCommand**](https://learn.microsoft.com/intune/configmgr/osd/understand/task-sequence-variables#OSDDoNotLogCommand)
    - Value: **true**

- **Set Task Sequence Variable**
    - Name: **Set Supervisor Password Variable**
    - Name: **SVP**
    - Tick *Do not display this value*
    - Value: Enter the BIOS supervisor password

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image9.png)

- **Run Command Line** - This command invokes **Set-LnvBiosCertificate** to provision the certificate to the device. The -Password parameter value uses the %SVP% task sequence variable set in the **Set Supervisor Password Variable** step.
    - Tick **Package** and choose the Lenovo BIOS Certificate Package created earlier

```dos
powershell.exe -ExecutionPolicy Bypass -Command "Set-LnvBiosCertificate -Certfile .\biosCert.pem -Password %SVP%"
```

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image10.png)

The last step is to restart the computer

## Monitoring the Results

Tracing the smsts.log, we see the Supervisor Password has been masked due to setting the **OSDDoNotLogCommand** variable to true.

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image11.png)

The output of provisioning the certificate to the device shows success.

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image12.png)

## User Experience

User is presented with a reboot message stating the system has converted to certificate-based authentication for BIOS access.

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image13.png)

Once the device restarts, you will see the configuration has changed and a 2nd reboot takes place.

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image14.png)

Accessing BIOS now requires an unlock code.

![](https://cdrt.github.io/mk_blog/img/2025/configmgr_deploy_cert_based_bios_auth/image15.jpg)
