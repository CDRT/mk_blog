---
date:
    created: 2025-09-05
authors:
    - Thad
categories:
    - "2025"
title: "Windows 11 ISO - Automate Partition Sizes for OEM Tooling"
---

# Automating Partition Sizes When Installing Windows From An ISO

As more corporate customers are moving to Intune and other cloud based tools for device management, there has been and probably will continue to be a decrease in the use of imaging technologies to deploy operating systems to devices.  Most of the time, customers are using the OEM provided preload on the device as the initial operating system installation.  Customers then leverage tools to continue with device configuration until the desired state is met.  

Using the OEM provided preload is an excellent choice for the initial operating system installation, but what happens when there is a need to reload Windows and there is no MDT or Configuration Manager to rely on?  The answer is to use the Windows ISO to load Windows on the device.  Customers have noted that when using the Windows 11 ISO to install Windows, they cannot update BIOS/UEFI and or Firmware on the device.  The following questions arise when we discuss utilizing the Windows 11 ISO to install Windows.

- How does an IT department standardize the drive configuration?
- What is the best method to guarantee enough space is allocated to the EFI System Partition (ESP) for booting the device and OEM tooling?
- Can we dynamically size the Operating System partition to maximize the drive space available to the end user?
- How do we guarantee the Recovery partition is the last partition on the drive?
- Can we prevent the Recovery partition from being grossly over provisioned?

The solution to all of these questions is fairly simple and does not require specialized tools to implement with exception of the Windows ADK being installed to a system.

# The Solution

To solve the above scenario, we can leverage the Windows 11 ISO and include an AutoUnattend.xml file, a start-configuration.bat file to call a diskpart script, and the configure-partitions.txt diskpart script file to configure drive.  The AutoUnattend.xml will define the disk and partition to install the operating sytem as well as directing setup to execute the batch file prior to running the Setup Wizard.  The executed batch file will first ask for a confirmation to erase the disk and, upon receiving the acknowledgement, will call the diskpart script to format and partition disk 0.  The diskpart script information will configure the disk as follows:
- Partition 1 set as an EFI System Partition (ESP) formatted as FAT32 with a size of 500 MB and has a label of System.
- Partition 2 set as a Microsoft System Reserved (MSR) Partition with a size of 16 MB
- Partition 3 set as a Primary Partition formatted as NTFS with a size set dynamically and has a label of Windows
- Partition 4 set as a Recovery Partition formatted as NTFS with a size of 990 MB and a label of Recovery

The goal is to meet OEM space needs, meet Windows requirements by exceeding the Microsoft minimums, and by the end of the task, still provision enough space for end users to work effectively.

### Additional Partitioning Information

After researching, we have found the EFI System Partition size should be more than the 200 MB defined by the Microsoft default configuration information found [here](https://learn.microsoft.com/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions?view=windows-11).  Beyond Microsoft using the EFI System Partition to boot the device, OEMs leverage this partition, hosting files used to update the BIOS/UEFI as well as system and component firmware.  Due to the size of BIOS/UEFI and Firmware updates, the default 100 MB EFI System Partition size can be quickly be exceeded, which is why we recommend the size increase.

For the Windows Primary Partition, we essentially allow diskpart to partition the remaining free space on the drive.  Once the partition is created, we shrink that partition down by the amount of space we want to allocate to the Recovery Partition, which in this case configured to occupy the last 990 MB on the disk.

## The Preparation

1. Create the following folder structure.  The structure listed will correlate to the steps in the [[#Building the ISO]] section.
    - C:\W11
    - C:\Mount
    - C:\ExtraFiles
2. Using the AutoUnattend.xml, Start-Configuration.bat, and Configure-Partitions.txt code at the end of this article, create each file in the C:\ExtraFiles directory.
3. Obtain the Windows 11 ISO file.
4. Right click on the Windows 11 ISO file and choose the Mount option.
5. Copy the entire folder structure and files from the mounted ISO to the C:\W11 directory.
6. [Download](https://learn.microsoft.com/windows-hardware/get-started/adk-install) and install the latest Windows ADK.  This installation is required for access to the OSCDIMG.exe to create the updated ISO file.

## Building the ISO 

1. Open Start > Windows Kits > Deployment and Imaging Tools Environment.  Be sure to run this using the Run As Administrator option.  To do this, right click on the Deployment and Imaging Tools Environement and choose "More >" and then Run As Administrator. 
2. Mount boot.wim index:2
    a. dism /Mount-Wim /WimFile:C:\w11\sources\boot.wim /index:2 /MountDir:C:\Mount
3. Copy the start-configuration.bat and configure-partitions.txt files from C:\ExtraFiles to C:\Mount\Windows\System32
4. Unmount boot.wim index:2
    a. dism /unmount-wim /mountdir:C:\Mount /commit
5. Copy the autounattend.xml file from C:\ExtraFiles to C:\W11
6. Create new ISO
    a. oscdimg -lW11 -m -u2 -bC:\W11\efi\microsoft\boot\efisys.bin C:\W11\ C:\W11_Prompt.iso

!!! info ""
    To remove the "Press any key to boot from CD/DVD..." prompt, change the oscdimg commmand to the following:
    oscdimg -lW11 -m -u2 -bC:\W11\efi\microsoft\boot\efisys_noprompt.bin C:\W11\ C:\W11_NoPrompt.iso

## Using the ISO

1. Using your favorite tool, apply the newly created ISO file to a USB drive.
2. Plug the USB drive into the computer
3. Power on the computer, enter the boot menu, and select the boot option for the USB drive.
4. If needed, press any key to continue the boot process.
5. At the command prompt, read the message and choose Y to erase the disk and use the partitions steps provided.  If you do not want to erase the disk, press N and the device will shutdown.
6. After the formatting and partitioning of the disk has completed, navigate through the Setup Wizard, making appropriate choices.
7. Once you finish the wizard, the device will begin to install Windows and will stop at OOBE.

## Scripts

### AutoUnattend.xml

```XML
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="windowsPE">
        <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <DiskConfiguration>
                <WillShowUI>OnError</WillShowUI>
            </DiskConfiguration>
        <RunSynchronous>
        <RunSynchronousCommand wcm:action="add">
                    <Path>cmd.exe /c "%systemroot%\System32\start-configuration.bat"</Path>
                    <Description>Initiate Disk and Partition Configuration</Description>
                    <Order>1</Order>
        </RunSynchronousCommand>
        </RunSynchronous>
        <ImageInstall>
                <OSImage>
                    <InstallTo>
                        <DiskID>0</DiskID>
                        <PartitionID>3</PartitionID>
                    </InstallTo>
                </OSImage>
            </ImageInstall>
        </component>
    </settings>
</unattend>
```

### Start-Configuration.bat

```CMD
@echo off
setlocal
:PROMPT
Echo This process is destructive and will wipe the disk. All data on the drive will be lost.
Echo Press Y to wipe the drive and use the preconfigured settings.
Echo Press N to discontinue the process and shutdown the computer.

SET /P AREYOUSURE=Do you want to continue (Y/[N])?
IF /I "%AREYOUSURE%" EQU "Y" GOTO CONFIGURE
wpeutil shutdown

:CONFIGURE
diskpart /s "%systemroot%\System32\configure-partitions.txt"
endlocal
```

### Configure-Partitions.txt

```TXT
rem === disk configuration ===
select disk 0
clean
convert gpt
rem === EFI Partition ===
create partition efi size 500
format quick fs=fat32 label="System"
assign letter=S
rem === MSR Partition ===
create partition msr size=16
rem === Windows Primary Partition ===
create partition primary
shrink minimum=990
format quick fs=ntfs label="Windows"
assign letter=W
rem === Recovery Partition ===
create partition primary size=990
format quick fs=ntfs label="Recovery"
set id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
gpt attributes=0x8000000000000001
assign letter=R
```
