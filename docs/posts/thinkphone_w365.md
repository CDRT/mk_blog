---
date: 2023-10-25
authors:
    - Phil
categories:
    - "2023"
title: Windows 365 and ThinkPhone
---

![W365](img\2023\thinkphone_w365\w365.jpg)
![Moto](img\2023\thinkphone_w365\motorola.png)

This article will serve as a basic walkthrough for bringing the full Windows 365 experience to your [Lenovo ThinkPhone](https://motorolanews.com/motorola-partners-with-microsoft-to-bring-new-productivity-features-to-the-thinkphone/).

"... The power and security of the Microsoft cloud with the versatility and simplicity of the PC" is now in your pocket!
<!-- more -->
## Prerequisites

To avoid redundancy or confusion, most of the steps laid out in this guide will link to Microsoft's documentation. It's assumed the reader has an active Azure subscription and working tenant with the necessary licenses ([Intune user](https://learn.microsoft.com/mem/intune/fundamentals/licenses-assign) and [Windows 365](https://learn.microsoft.com/windows-365/business-enterprise-comparison#purchasing-and-licensing-comparisons)) assigned.

ThinkPhone can connect to either Windows 365 Enterprise or Business cloud PCs. Refer to the edition comparisons [here](https://learn.microsoft.com/windows-365/business-enterprise-comparison).

Ensure your tenant is configured for user and device [enrollment](https://learn.microsoft.com/mem/intune/fundamentals/deployment-guide-enrollment) and verify the cloud PC has been [provisioned](https://learn.microsoft.com/windows-365/enterprise/deployment-overview) successfully (if deploying an Enterprise SKU).

## Enrolling ThinkPhone in Intune

There are several options when [enrolling Android devices](https://learn.microsoft.com/mem/intune/fundamentals/deployment-guide-enrollment-android). For this guide, I'm going to enroll my ThinkPhone as an [Android Enterprise fully managed device](https://learn.microsoft.com/mem/intune/fundamentals/deployment-guide-enrollment-android#android-enterprise-fully-managed) (COBO).

Be sure to complete the [task list](https://learn.microsoft.com/mem/intune/fundamentals/deployment-guide-enrollment-android#admin-tasks-fully-managed). Once your [enrollment profile](https://learn.microsoft.com/mem/intune/enrollment/android-fully-managed-enroll#step-2-create-new-enrollment-profile) has been created, you're ready to power on your ThinkPhone and bring it under management.

I'm going to enroll using a [QR code](https://learn.microsoft.com/mem/intune/enrollment/android-dedicated-devices-fully-managed-enroll#enroll-by-using-a-qr-code). Once the QR reader is open, scan the enrollment profile QR code

!!! info ""
    For demo purposes, this is what the QR/token looks like and will be revoked after this blog is published

![QR](img\2023\thinkphone_w365\image1.jpg)

Next, connect to a Wi-Fi network and follow the many on-screen prompts.

!!! info ""
    There's at least a dozen different screens to transition through during enrollment so I'm only adding a handful of the important ones

![Setup](img\2023\thinkphone_w365\image2.jpg)

![Setup](img\2023\thinkphone_w365\image3.jpg)

![Setup](img\2023\thinkphone_w365\image4.jpg)

![Setup](img\2023\thinkphone_w365\image5.jpg)

## Connecting to Windows 365

Upon successful enrollment, pair your ThinkPhone to your Bluetooth keyboard/mouse and connect to a monitor via USB-C cable. Moto Connect will extend to your monitor and what's first in the list of apps to choose? Windows 365?!

![Connect](img\2023\thinkphone_w365\image6.jpg)

Click on Windows 365. Notice the screen change to the Windows 365 wallpaper and on your phone, choose which cloud PC to connect to

![Connect](img\2023\thinkphone_w365\image7.jpg)
![Connect](img\2023\thinkphone_w365\image8.jpg)

Enter your credentials and you'll be taken to your Windows 365 cloud PC.

![Connect](img\2023\thinkphone_w365\image9.jpg)
![Connect](img\2023\thinkphone_w365\image10.jpg)

## Deploying the Motorola Ready For Assistant

To take advantage of the enhanced ThinkPhone experience when combined with a Lenovo Windows PC, you can deploy the Ready For Assistant store app to your users' devices.

Login to the [Intune admin center](https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/AppsWindowsMenu/~/windowsApps)

Go to the Apps blade. Click **+Add** to create a new app.

Select **Microsoft Store app (new)** for App type and click **Select**
![Add App](img\2023\thinkphone_w365\image13.png)

Click **Search the Microsoft Store app (new)**. If you search the store for **Ready For Assistant**, you won't get any results. However, you can search using the app ID **XP8JRF5SXV03ZM** to add it

![Find App](img\2023\thinkphone_w365\image14.png)

Click on the app and then click **Select**. Once the app is added, find it in the list and open its Properties.

![App Properties](img\2023\thinkphone_w365\image15.png)

Scroll to the bottom and click **Edit** next to Assignments

![Edit Assignment](img\2023\thinkphone_w365\image16.png)

Configure the assignments as desired and then click **Review + save**

![Assign App](img\2023\thinkphone_w365\image17.png)

Devices will then get the app based on the configuration of the Assignment. It may require the devices to get to their scheduled sync before the offer is made.

You can force a sync by going to the device in the Devices view and clicking the ![Sync](img\2023\thinkphone_w365\image18.png) button.
