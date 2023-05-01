---
layout: post
---
![Installing WSA](assets/wsa/wsa_install.png)

[Windows Subsystem for Android](https://learn.microsoft.com/en-us/windows/android/wsa/) (WSA) is a program that allows users to run Android applications on Windows by utilizing Microsoft's Hyper-V virtualization technology. Now suspects of criminal investigations may have evidence that is traditionally found on their phone inside of their computer. This creates a need for a  new form of forensic acquisitions that blends both mobile and desktop techniques and tools. In this blog post, I will be exploring WSA -- from reverse engineering core system executable to exploring strange directory paths for forensic artifacts.

> Note: WSA is only officially supported on Windows 11 and requires Hyper-V. This means you will not be able to use WSA in a Virtual Machine (VM) unless you have correctly nested the virtualization technology.

---

# Table of Contents
1. [Getting Familiar With WSA](#Getting-Familiar-With-WSA)
      1. [Prerequisites](#Prerequisites)
      2. [Downloading WSA](#Downloading-WSA)
      3. [WSA MSIX Package](#WSA-MSIX-Package)
      4. [WSA Executables](#WSA-Executables)
2. [Forensic Artifacts](#Forensic-Artifacts)
      1. [Artifacts of installing WSA](#IArtifacts-of-installing-WSA)
      2. [Artifacts of installing a WSA App](#Artifacts-of-installing-a-WSA-App)
      3. [Artifacts of WSA App Activity](#Artifacts-of-WSA-App-Activity)
            - [WSA VHDX Files](#WSA-VHDX-Files)
3. [Processing a Forensic Image with WSA](#Processing-a-Forensic-Image-with-WSA)

---

# Getting Familiar With WSA

## Prerequisites
According to [Microsoft's documentation](https://support.microsoft.com/en-us/windows/install-mobile-apps-and-the-amazon-appstore-on-windows-f8d0abb5-44ad-47d8-b9fb-ad6b1459ff6c), your system must meet the following requirements:
- Windows 11
- 8 GB RAM
- x64 or ARM64 Processor Architecture
- Must enable Virtual Machine Platform feature

![Enabling VMP](assets/wsa/windows_features.png)

## Downloading WSA
Before we can begin our analysis of WSA, we must first obtain it. WSA can be downloaded and installed in one of two ways:

1. Install the [Amazon Appstore](https://apps.microsoft.com/store/detail/amazon-appstore/9NJHK44TTKSX) (this method required an Amazon account)
2. Manually obtain the WSA MSIX Package
    1. GoTo: [https://store.rg-adguard.net/](https://store.rg-adguard.net/)
    2. Select **Productid** and enter `9p3395vx91nr`, then select **Slow**
    3. Click on the link: **MicrosoftCorporationII.WindowsSubsystemForAndroid_2210.40000.7.0_neutral_~_8wekyb3d8bbwe.msixbundle**
    4. Install the package by double clicking or running the following PowerShell command: 

           Add-AppxPackage MicrosoftCorporationII.WindowsSubsystemForAndroid_2210.40000.7.0_neutral_~_8wekyb3d8bbwe.msixbundle.msixbundle

![Downloading WSA](assets/wsa/wsa_download.png)

## WSA MSIX Package

If you manually downloaded the WSA MSIXbundle file, then you have an opportunity to examine how WSA is structured before installing it. According to [Microsoft's documentation](https://learn.microsoft.com/en-us/windows/msix/overview), an MSIX file is "a Windows app package format that provides a modern packaging experience to all Windows apps." We can use a tool such as [MSIX Hero](https://msixhero.net/) to examine the internal structure of this mysterious MSIX file.

First, we extract the MSIXbundle file using an archiving tool such as [7-zip](https://www.7-zip.org/). This should leave us with a folder containing a few dozen MSIX files along with a folder named `AppxMetdata` which contains the file `AppxBundleManifest.xml`. . The MSIX file that we want to examine will depend on our CPU architecture:
- `WsaPackage_2302.40000.9.0_x64_Release-Nightly.msix`
- `WsaPackage_2302.40000.9.0_ARM64_Release-Nightly.msix`

Let's take a look at the x64 package using MSIX Hero.

![WSA MSIX Package](assets/wsa/wsa_msix.png)

## WSA Executables

- WSA Settings Application: `C:\Program Files\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_2303.40000.5.0_x64__8wekyb3d8bbwe\WsaSettings.exe`

   - WSA Client: `C:\Program Files\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_2303.40000.5.0_x64__8wekyb3d8bbwe\WsaClient\WsaClient.exe`

      - WSA GSK Server: `C:\Program Files\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_2303.40000.5.0_x64__8wekyb3d8bbwe\GSKServer\GSKServer.exe`

- WSA Service Manager: `C:\Program Files\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_2303.40000.5.0_x64__8wekyb3d8bbwe\WsaService\WsaService.exe`

Reverse engineering WSAClient.exe reveals support for the following arguments:
- /restart
- /launch  (For example: `/launch wsa://com.android.documentsui`)
- /modify
- /deeplink
- /uninstall
- /shutdown
- /toast

### WSA Virtual Machine Processes
| Program Name | Description |
| ----------- | ----------- |
| vmcompute.exe | Hyper-V Host Compute Service |
| vmwp.exe | Virtual Machine Worker Process |
| vmmemWSA | WSA VMMWP Instance |

---

# Forensic Artifacts

## Artifacts of installing WSA

By running [Regshot](https://github.com/Seabreg/Regshot) and [Procmon](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) during the installation of the WSA package, created and modified directories and registry keys can be logged. 

Installation of WSA creates a number of key directories:

| Path | Description |
| --- | ----------- |
| `C:\Program Files\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_2301.40000.7.0_x64__8wekyb3d8bbwe` | Core of WSA. All of the WSA executables are stored here. |
| `C:\Users\[USERNAME]\AppData\Local\Packages\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe` | WSA user data including Android VHDX. |

## Artifacts of installing a WSA App

An app can be installed in one of two ways:
1. Through the Amazon AppStore
2. Manually installing an APK

When an app is installed into WSA, the host Windows operating system is notified and generates a number of helpful artifacts:
- A .lnk file is generated in the directory
`C:\Users\acbuc\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\` that contains a link to launch the app using the WSAClient.exe:

      C:\Users\acbuc\AppData\Local\Microsoft\WindowsApps\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe\WsaClient.exe /launch wsa://uk.co.aifactory.chessfree

> Note: This link allows Windows to display the app in the Windows start menu, appearing as though it is a native system application.

- The Image icons for the installed apps are stored in:
`C:\Users\acbuc\AppData\Local\Packages\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe\LocalState\[appID].ico/.png`

![WSA App Icons](assets/wsa/wsa_app_icons.png)

## Artifacts of WSA App Activity

### WSA VHDX Files

Since the WSA is essentially a virtual machine, there must be a virtual hard disk where it stores all the Android system files.
These virtual hard disk files are stored in `C:\Users\acbuc\AppData\Local\Packages\MicrosoftCorporationII.WindowsSubsystemForAndroid_8wekyb3d8bbwe\LocalCache`

![WSA VHDX Files](assets/wsa/vhdx_files.png)

A VHDX file is a Microsoft virtual hard disk format used by Hyper-V. Normally, a VHDX file may be opened or mounted to view the contents of the virtual filesystem. Windows Subsystem for Linux (WSL) uses a VHDX file to store its virtual filesystem. Although the WSL VHDX file can be easily opened, the WSA VHDX file crashes FTK Imager and other software attempting to open it.

![VHDX Encrypted](assets/wsa/vhdx_encrypted.png)

It appears Microsoft is either encrypting or compressing the VHDX file for WSA. Let's examine the [VHDX specification sheet](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-vhdx/83e061f8-f6e2-4de1-91bd-5d518a43d477) and see if we can determine what makes this VHDX file special. Let's open the VHDX file with the [HxD hex editor](https://mh-nexus.de/en/hxd/) and start prasing the format.

![VHDX Headers](assets/wsa/vhdx_headers.png)

It appears `LogOffset` is not properly pointing to the corret offset of the log section. It is located at `0x101000` instead of `0x100000`. Let's patch the file by changing the offset to the correct value (in little endianess), and recalculating the CRC-32C checksums (setting the CRC 4 byte field to 00's during the calculation). Now we can attempted to read the VHDX files and... 

It didn't work.

It looks like decoding the VHDX format used for WSA may involve reverse engineering the WSA executables that load the VHDX files into memory before starting the virtual system. Properly decoding this VHDX file may be the topic of a future blog posh. For now, let's try simply importing a VHDX file from another WSA instance into our own local WSA installation.

# Processing a Forensic Image with WSA

Forensics expert Jessica Hyde has kindly provided me with a sample forensic image of a device where the user has installed and interacted the Windows Subsystem for Android. Let's open the image up in [FTK Imager](https://www.exterro.com/ftk-imager) and see if we can figure out what the user was doing in WSA using the information we have learned.

