---
date: 
    created: 2023-07-24
authors: 
    - Joe
    - Devin
categories:
    - "2023"
title: Certificate-based BIOS Authentication
cover_image: \img\2023\cert_based_bios_authentication\signed_settings.png
---

Beginning with 2022 ThinkPad models, it is now possible to configure systems to use a digital signing certificate instead of a supervisor password. Although this feature does not eliminate the challenge of initially securing the device, it does eliminate the need to exchange passwords in plain text when scripting BIOS settings changes.
<!-- more -->
To leverage this capability, a new set of WMI methods were introduced as part of the Lenovo WMI BIOS Interface. The usage concept follows these steps:

1. Create a code signing certificate with public key/private key files
1. Provision devices with the certificate
1. Generate a "signed settings string"
1. Call the new method to set the setting on the target device
1. Repeat steps 3 & 4 as needed

To make the process easier, we have created a LnvBiosCerts PowerShell module and the Lenovo BIOS Cert Tool (LnvBiosCertsInterface.ps1) which provides a GUI interface for working with these functions. The current version of this package can be downloaded from [here (LnvBiosCertTool_1.0.2.zip)](https://download.lenovo.com/cdrt/tools/LnvBiosCertTool_1.0.2.zip).

A reference guide to the PowerShell module is available at [https://docs.lenovocdrt.com/guides/lbct/index](https://docs.lenovocdrt.com/guides/lbct/index)

## Getting Started

The first step in trying this out will be to generate a compatible code signing certificate. In this article we will use OpenSSL to create the certificate. In this guide we will be using OpenSSL Light for this. This can be installed on an administrator’s system using Winget:

![Install OpenSSL Light](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\installopenssl.png)

Once it is installed, add the path to the CLI tool to the Path environment variable.

```PowerShell
$env:Path += ";C:\Program Files\OpenSSL-Win64\bin"
```

Then run the following command to generate a private key as a .PEM file:

```PowerShell
openssl genrsa -out privateKey.pem 2048
```

![Create certificate](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\createkey.png)

This file should be kept secure with access only given to an administrator. Having access to this file would be analogous to knowing a Supervisor Password.

Depending on your company's requirements you may need to use .DER file format instead.  These are also supported and can be generated from the .PEM file. This solution does not currently support encrypted keys.

The optional command to convert the .PEM file to a .DER file:

```PowerShell
openssl rsa -in privateKey.pem -inform PEM -out privateKey.der -outform DER
```

A public key can be generated from the private key with the following command:

```PowerShell
openssl rsa -in privateKey.pem -pubout -out publicKey.pem
```

Convert to .DER (optional):

```PowerShell
openssl rsa -pubin -in publicKey.pem -inform PEM -out publicKey.der -outform DER
```

Next we need to generate the X509 Certificate that will be provisioned to the managed devices.

```PowerShell
openssl req -new -x509 -days 7300 -key privateKey.pem -out biosCert.pem -sha256 [-config openssl.cnf]
```

!!! info ""
    If you installed the Lite version of OpenSSL for Windows, the config file will not be there so you can ignore the -config openssl.cnf parameter.

![Create certificate](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\createbioscert.png)

## Deploy the LnvBiosCerts Module

The zip file linked at the beginning of this article contains a subfolder named LnvBiosCerts which contains the PowerShell module. To install the module, you can simply copy that folder to one of the following locations:

```C:\Program Files\WindowsPowerShell\Modules```

or

```C:\Users\<username>\Documents\WindowsPowerShell\Modules```

The first location is preferable as it allows any user or admin process on the machine to access the module.  The second location only makes the module available for that particular user.

![PowerShell Modules Folder](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\screen1.png)

Open a new Administrator PowerShell terminal. This ensures the module just copied to this system will be loaded. If, however, the commands below are not recognized you may need to explicitly import the module with this command:

```PowerShell
Import-Module 'LnvBiosCerts' -force
```

Run the following command in the folder where the **biosCert.pem** file is located to provision the certificate to the system:

```PowerShell
Set-LnvBiosCertificate -Certfile .\biosCert.pem -Password pass1word
```

![PowerShell Modules Folder](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\installcert.png)

!!! info ""
    The -Password parameter passes the current Supervisor Password that exists on the device and this password will be removed and replaced with the certificate. The target device must either have a Supervisor Password already set or must be in the System Deployment Boot Mode with the command being run from a script under WinPE of a PXE boot image.

Reboot the system so that the changes can be finalized.  You may notice a message during reboot that confirms the configuration has changed.

## Generate a Signed Command

On the administrator device which has the private key, generate a signed command.  This can be done using the Lenovo BIOS Certificate Tool user interface by doing the following:

1. Click Generate Signed Command in the left navigation menu
1. Select the method ‘SetBiosSetting’
1. Specify the path to the private key
1. Either enter or select the setting name and value (use the Load WMI Settings button if the current system has the setting needed, otherwise you can type the setting name and value in manually)
1. Click Generate Command
1. Copy the text generated to a text file you can copy to or access from the test device

The signed command can also be generated from the PowerShell terminal by running the following command in the folder where the private key exists:

```PowerShell
Get-LnvSignedWmiCommand -Method SetBiosSetting -SettingName WakeOnLANDock -SettingValue Disable -KeyFile .\privateKey.pem | Out-File .\setting.txt
```

This will generate a text file for you containing the signed command.

<!-- 
![Generate signed command](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\generatesignedcommand.png)
-->

!!! info ""
    You can generate multiple signed commands to change multiple BIOS Settings. You must also create a signed command that uses the SaveBiosSettings method.  This must be the final command submitted after changing one or more settings to ensure the settings are saved prior to restarting the machine.

## Apply the Signed Command

On the test machine where the certificate has been applied and the LnvBiosCerts module has been installed, run the following command to apply the signed command which can be copied from the text file created in the previous step:

```PowerShell
Submit-LnvBiosChange -Command “(text from text file)”
```

![Apply signed command](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\applycommand.png)

Repeat this step for all the signed commands and make sure the last one applied is the one that references the SaveBiosSettings method. Restart the system for the settings changes to take effect.

## Going Further

The Lenovo BIOS Cert Tool provides an easy to use graphical interface to work with the certificate-based BIOS configuration methods. There are several additional functions provided by the Lenovo BIOS Cert Tool that are described below.

### Working with Think BIOS Config Tool

The Think BIOS Config Tool is a separate tool that provides the ability to list all the available settings and values on a system. The settings can be configured as desired and an INI file can be generated with changed settings. The Lenovo BIOS Certificate Tool can convert one of these INI files into a text file containing the signed WMI commands to apply the settings.

![Convert Config File Window](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\convertconfigfilewindow.png)

The generated file can be used on the ***Apply Signed Commands*** page of the Lenovo BIOS Certificate Tool or can easily be incorporated into your own PowerShell script.

![Apply Signed Commands Window](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\applysignedcommandwindow.png)

### Unlocking System Secured with Certificate

When a system is secured with a certificate, you cannot access BIOS Setup directly on the machine using a Supervisor Password. Therefore, there is a different process to unlock a machine to access BIOS Setup.

1. Technician boots the system pressing F1 to enter BIOS Setup.
1. Insert a USB key to save a BIOS Unlock File to
1. Click "Generate Unlock Code"
1. Send BIOS Unlock File to administrator
1. Administrator uses Lenovo BIOS Cert Tool to generate Unlock Code by specifiying Private Key File and Unlock File.
1. Administrator sends Unlock Code to Technician
1. Technician enters Unlock Code and continues to BIOS Setup

![Unlock BIOS Window](https://cdrt.github.io/mk_blog/img\2023\cert_based_bios_authentication\unlockbioswindow.png)

## Final Notes

### Clearing certificate

If you are clearing the certificate from multiple machines, it makes more sense to change the certificate to a temporary password instead of just removing the certificate. Generating a **ClearBiosCertificate** command requires the machine serial number which means there is a unique command for each machine.

**ChangeBiosCertificateToPassword** would be a general command for all machines with that certificate which will replace the certificate with a password you specify. This will leave the devices in a secure, managable state also.

### Installing Module to auto load

By placing the **LnvBiosCerts** folder in the **%Program Files%\WindowsPowershell\Modules** folder, the module will auto load when a PowerShell window is started.

### Limitations

The **SetFunctionRequest** method has a limitation regarding the ***ResetSystemToFactoryDefaults*** BIOS function. The method will not be able to call this function with a certificate installed on the machine.
