---
layout: post
---
![Installing WSA](/assets/wsa/wsa_install.png)

[Windows Subsystem for Android](https://learn.microsoft.com/en-us/windows/android/wsa/) (WSA) is a program that allows users to run Android applications on Windows by utilizing Microsoft's Hyper-V virtualization technology. This creates a need for a  new form of forensic acquisitions that blend both mobile and desktop techniques and tools. In this blog post, I will be exploring WSA; from reverse engineering core system executable, to exploring strange directory paths for forensic artifacts.

> Note: WSA is only officially supported on Windows 11 and requires Hyper-V. This means you will not be able to use WSA in a VM unless you have correctly nested the virtualization technology.

## Prerequisites
According to [Microsoft's documentation](https://support.microsoft.com/en-us/windows/install-mobile-apps-and-the-amazon-appstore-on-windows-f8d0abb5-44ad-47d8-b9fb-ad6b1459ff6c), your system must meet the following requirements:
- Windows 11
- 8 GB RAM
- x64 or ARM64 Processor Architecture
- Must enable Virtual Machine Platform feature

![Enabling VMP](/assets/wsa/windows_features.png)

## Downloading WSA
Before we can begin our analysis of WSA, we must first obtain it. WSA can be downloaded and installed in one of two ways:

1. Install the [Amazon Appstore](https://apps.microsoft.com/store/detail/amazon-appstore/9NJHK44TTKSX) (this method required an Amazon account)
2. Manually obtain the WSA MSIX Pacakge
    1. GoTo: [https://store.rg-adguard.net/](https://store.rg-adguard.net/)
    2. Select **Productid** and enter `9p3395vx91nr`, then select **Slow**
    3. Click on the link: **MicrosoftCorporationII.WindowsSubsystemForAndroid_2210.40000.7.0_neutral_~_8wekyb3d8bbwe.msixbundle**
    4. Install the package by double clicking or running the following PowerShell command: 

           Add-AppxPackage MicrosoftCorporationII.WindowsSubsystemForAndroid_2210.40000.7.0_neutral_~_8wekyb3d8bbwe.msixbundle.msixbundle

![Downloading WSA](/assets/wsa/wsa_download.png)

## WSA MSIX Package

If you manually downloaded the WSA MSIXbundle file, then you have an oppertunity to examine how WSA is structures before actually installing it. According to [Microsoft's documentation](https://learn.microsoft.com/en-us/windows/msix/overview), an MSIX file is "a Windows app package format that provides a modern packaging experience to all Windows apps." We can use a tool such as [MSIX Hero](https://msixhero.net/) to examine the internal strucutre of this mysterious MSIX file.

First We extract the MSIXbundle file using an archiving tool such as [7-zip](https://www.7-zip.org/). This should leave us with a folder containing a few dozen MSIX files along with a folder named `AppxMetdata` which contains the file `AppxBundleManifest.xml`. . The MSIX file that we want to examine will depend on our CPU architecture:
- `WsaPackage_2302.40000.9.0_x64_Release-Nightly.msix`
- `WsaPackage_2302.40000.9.0_ARM64_Release-Nightly.msix`

Let's take a look at the x64 package using MSIX Hero.

![WSA MSIX Package](/assets/wsa/wsa_msix.png)


## WSA Installation

During the installation of the WSA package, a number of key directories are created:

`C:\Program Files\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_2301.40000.7.0_x64__8wekyb3d8bbwe`

This location is the core of WSA. All of the WSA executables are stored here.

`C:\Users\[USERNAME]\AppData\Local\Packages\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe`

This is where all of the interesting user artifacts are stores.

## Running WSA



## Installing an App

An app can be installed in one of two ways:
1. Through the Amazon AppStore
2. Manually installing an APK

When an app is installed into WSA, the host Windows operating system is notified creates a number of artifacts:
- A .lnk file is generated in the directory
`C:\Users\acbuc\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\` that contains a link to launch the app using the WSAClient.exe:

      C:\Users\acbuc\AppData\Local\Microsoft\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe\WsaClient.exe /launch wsa://uk.co.aifactory.chessfree

> Note: This link allows Windows to display the app in the Windows start menu, appearing as though it is a native system application.

- The Image icons for the installed apps are stored in:
`C:\Users\acbuc\AppData\Local\Packages\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe\LocalState\[appID].ico/.png`

![WSA App Icons](/assets/wsa/wsa_app_icons.png)