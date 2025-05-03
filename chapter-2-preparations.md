---
title: Deploy HANA Express 2.0 on Microsoft Hyper-V - Chapter 2
author: st-gr
date: 5/1/2025
mainfont: Helvetica, Arial, sans-serif
fontsize: 18px
---

SAP HANA Express 2.0 on Hyper-V
===============================

***Guide to deploy HANA Express 2.0 on Microsoft Hyper-V***

**Author:** *st-gr*

[<< Previous Chapter](chapter-1-motivation.md) | [Content Table](README.md) | [Next Chapter >>](chapter-3-convert-appliance.md)

---

# Chapter Two: Preparations

## Get SAP HANA Express

Go to [SAP HANA trial][1] and get yourself a download of the SAP HANA Express 2.0 appliance. 

![Provide email address](/assets/access-sap-trial.png)

Obviously you have read through and accepted the terms and conditions. In a gist:
* You are licensed for 64 GB and your license ends after 12 months.
* The bundled SUSE Linux Enterprise Server may only be used with SAP HANA express edition, general purpose usage of the OS is prohibited.
* More terms like telemetry data collection: In your own interest  make it a habit to read any fine-print you are presented with these days.

Your computer also needs to meet the hardware requirements to run the appliance. I will use 32 GB of RAM for my purposes, you can get by with 8 GB RAM for the data base server only or 16 GB for the database + XS Advanced applications.

### Download

You are presented with a download links page. There you are informed that you need to have a Java Runtime environment pre-installed.
The provided link to [tools.hana.ondemand.com][2] has a long list of tools. Search for SAP JVM. I however installed [SapMachine][3] on my computer.

![Download links page](/assets/sap-download-links-page.png)

### SAP Download Manager quirks
I bet you were tempted as I was to click on the *Windows DM* link. I realized quickly that with my JVM setup this App didn't start.
I then downloaded the *Platform-independent DM* and executed it in a PowerShell 5.1 terminal with:

    java -jar HXEDownloadManager.jar

I was presented with:

    Cannot open log file C:\Users\me\AppData\Local\Temp\1\\hxedm250501.log.

If you are detail oriented and know a little about how Windows pathnames are supposed to look like then the double backslash before the `hxedm*.log` sticks out to you as the culprit.

I will spare you having to disassemble the jar file to find which environment variable influences the download manager to use a different temporary folder.

Solution: Set the environment variable `TMP` to a temporary folder of your choosing. I used:

    set TMP=C:\Windows\Temp
    java -jar HXEDownloadManager.jar

Now the splash screen shows and the download manager launches.

![Download Manager launches](/assets/download-manager-starts-tmp-set.png)

For my purposes I chose to download the complete version of HANA Express which supports running Cloud Foundry based HANA XSA applications.

![Download Manager selected software to download](/assets/download-manager-selected-software.png)

Click download and wait until the (up to 38 GB) download completes.

## Other software to install

This guide assumes that you have the following software pre-installed:
* The before mentioned [SapMachine][3]
    * to execute the Download Manager
* [Microsoft Hyper-V][4]
    * to run the appliance
* [qemu-img][5] for Windows
    * qemu-img will help us convert the virtual hard drive image file to a format that Hyper-V supports
    * you can also use VBoxManage or the StarWind V2V converter
* [7-Zip][6]
    * 7-Zip will allow us to extract the .ova file later
* [COMpipe][7] (optional, download the x64/Release .exe)
    * Lets you use TeraTerm or PuTTY to connect to the VM shell
    * Then you can use copy and paste even when no network is connected

[1]: https://www.sap.com/products/data-cloud/hana/express-trial.html
[2]: https://tools.hana.ondemand.com/#cloud
[3]: https://sapmachine.io/
[4]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/install-hyper-v?pivots=windows
[5]: https://cloudbase.it/qemu-img-windows
[6]: https://www.7-zip.org/
[7]: https://github.com/tdhoward/COMpipe/tree/master

---

[<< Previous Chapter](chapter-1-motivation.md) | [Content Table](README.md) | [Next Chapter >>](chapter-3-convert-appliance.md)
