---
date:
    created: 2017-01-06
authors:
    - Phil
categories:
    - "2017"
title: Converting BIOS to UEFI on Think Products with ConfigMgr
---

A new Task Sequence Variable, **TSUEFIDrive**, was introduced in Configuration Manager Current Branch version 1610. This variable will prepare the hard drive for transition to UEFI from legacy BIOS, in one task sequence. This is extremely helpful if you're migrating systems from Windows 7 to Windows 10 in a refresh scenario.

A detailed walk-through by the Microsoft team on how to configure your Task Sequence for use with this variable can be found [**here**](https://docs.microsoft.com/sccm/osd/deploy-use/task-sequence-steps-to-manage-bios-to-uefi-conversion). We want to focus on step 5 from this guide:
<!-- more -->
!!! info ""
    Add a step to start the OEM tool that will convert the firmware from BIOS to UEFI. This will typically be a Run Command Line task sequence step with a command line to start the OEM tool.

This blog post will demonstrate how to accomplish this scenario using the [Think BIOS Config Tool](https://docs.lenovocdrt.com/#/tbct/tbct_top).

On your test ThinkPad or ThinkCentre, use the Think BIOS Config tool to configure the BIOS settings you want applied to the rest of your ThinkPads/ThinkCentres and export to an .ini.

Here is a sample .ini exported from a ThinkPad T460

```ini
WakeOnLAN,ACOnly
EthernetLANOptionROM,Enable
IPv4NetworkStack,Enable
IPv6NetworkStack,Enable
UefiPxeBootPriority,IPv4First
WiGigWake,Disable
USBBIOSSupport,Enable
AlwaysOnUSB,Enable
TrackPoint,Automatic
TouchPad,Automatic
FnCtrlKeySwap,Disable
FnSticky,Disable
FnKeyAsPrimary,Disable
BootDisplayDevice,LCD
SharedDisplayPriority,DockDisplay
TotalGraphicsMemory,256MB
BootTimeExtension,Disable
SpeedStep,Enable
AdaptiveThermalManagementAC,MaximizePerformance
AdaptiveThermalManagementBattery,Balanced
CPUPowerManagement,Automatic
OnByAcAttach,Disable
PasswordBeep,Disable
KeyboardBeep,Enable
AMTControl,Disable
LockBIOSSetting,Disable
MinimumPasswordLength,Disable
BIOSPasswordAtUnattendedBoot,Enable
BIOSPasswordAtReboot,Disable
BIOSPasswordAtBootDeviceList,Disable
PasswordCountExceededError,Enable
FingerprintPredesktopAuthentication,Enable
FingerprintReaderPriority,External
FingerprintSecurityMode,Normal
FingerprintPasswordAuthentication,Enable
SecurityChip,Active
TXTFeature,Disable
PhysicalPresenceForTpmProvision,Disable
PhysicalPresenceForTpmClear,Enable
BIOSUpdateByEndUsers,Enable
SecureRollBackPrevention,Disable
DataExecutionPrevention,Enable
VirtualizationTechnology,Enable
VTdFeature,Enable
EthernetLANAccess,Enable
WirelessLANAccess,Enable
WirelessWANAccess,Enable
BluetoothAccess,Enable
USBPortAccess,Enable
MemoryCardSlotAccess,Enable
IntegratedCameraAccess,Enable
MicrophoneAccess,Enable
FingerprintReaderAccess,Enable
NfcAccess,Enable
WiGig,Enable
BottomCoverTamperDetected,Disable
InternalStorageTamper,Disable
ComputraceModuleActivation,Enable
SecureBoot,Enable
SGXControl,SoftwareControl
BootMode,Quick
StartupOptionKeys,Enable
BootDeviceListF12Option,Enable
BootOrder,USBCD:USBFDD:NVMe0:HDD0:USBHDD:PCILAN
NetworkBoot,PCILAN
BootOrderLock,Disable
```

If you have a mix of Think products, you can combine all settings in a single .ini. If the setting does not apply to the client, it will simply be skipped. I chose the Virtualization settings for this example because all ThinkPads ship with these disabled. If you want to leverage Device Guard at some point, which requires virtualization to be enabled as a prerequisite, you can achieve this in the same step.

You can also reduce the .ini file down to just the lines containing the settings you care about. Here's an example of an .ini that contains settings that will be applied to both platforms:

```ini
VirtualizationTechnology,Enable
Intel(R) Virtualization Technology,Enabled
VTdFeature,Enable
VT-d,Enabled
SecureBoot,Enable
Secure Boot,Enabled 
```

!!! info ""
    Notice there are two of each setting being applied. This is because these values are worded differently between ThinkPad and ThinkCentre.

Create a new Package in your ConfigMgr console that contains the HTA and the .ini and distribute to your distribution points. Now, back to step 5 above, add a Run Command Line step to call the HTA and apply the .ini as shown below. The command line will work if no Supervisor password is set on the client.

!!! info ""
    HTA support will need to be added to the boot image

![](\img/2017/bios_to_uefi/image1.jpg)

```cmd
cmd.exe /c ThinkBiosConfig.hta "file=ThinkPadConfig.ini"
```

If you have a Supervisor password set on your clients, you can generate an encryption key in the BIOS Config Tool that will be used to pass the Supervisor password to set the BIOS changes. For more info, refer to the documentation. In the below screenshot, the command line now includes the "pass" switch with the encryption key I generated from the tool

![](\img/2017/bios_to_uefi/image2.jpg)

That's it! You'll notice the system restart twice for the changes to take effect but the task sequence will resume and boot into PE to finish out the deployment since the boot image was staged to the hard drive prior to restart.
