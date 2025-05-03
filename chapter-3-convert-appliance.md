---
title: Deploy HANA Express 2.0 on Microsoft Hyper-V - Chapter 3
author: st-gr
date: 5/1/2025
mainfont: Helvetica, Arial, sans-serif
fontsize: 18px
---

SAP HANA Express 2.0 on Hyper-V
===============================

***Guide to deploy HANA Express 2.0 on Microsoft Hyper-V***

**Author:** *st-gr*

[<< Previous Chapter](chapter-2-preparations.md) | [Content Table](README.md) | [Next Chapter >>](chapter-4-create-vm.md)

---

# Chapter Three: Convert the virtual appliance

Your `HXE` download folder should look similar to this:

![Folder containing the downloaded files - note that I also downloaded `qemu-img` there](/assets/hxe-downloads-folder.png)

## The .ova file

From a technical perspective the .ova file is a standardized
archive format similar to .zip. Let us look into it.

SHIFT-Right-click on it and chose the 7-Zip context menu item to open the archive or use any other way to inspect the ova contents.

![Virtual Appliance ova file opened in 7-Zip](/assets/ova-opened-in-7-zip.png)

We see that the ova archive contains another archive in the .gz format and as the name indicates it is a VMDK virtual hard disk image file.

### Extract the disk image

Extract the ova contents into a subfolder. I created the subfolder `hxesa` and then extracted the files there. Then I also used 7-Zip to extract the `hxexsa-disk1.vmdk.gz` file into yet another subfolder `hxexsa-disk1.vmdk`.

![Extracted VMDK hard disk image](/assets/extracted-vmdk-image.png)

## Convert VMDK hard disk image to VHDX format

Microsoft Hyper-V supports only VHD or VHDX virtual hard disk formats [VHD (file format)][8].
The VHDX format is preferred for Gen2 virtual machines that mandate to use UEFI bootloaders.

We target the VHD successor format which is VHDX.

### Qemu-Img to the rescue

Qemu is a fantastic piece of free software which was created by the hyper-productive French programmer [Fabrice Bellard][9] which also brought to you `ffmpeg` without which you won't be able to watch streaming TV, for example.

A tool which is part of the Qemu project is `qemu-img` which is capable to convert images between different formats. We will use it to convert the VMDK file to the Hyper-V compatible VHDX format.

Open a terminal (here PowerShell 5.1) in the folder where the VMDK was extracted to and enter:

    ..\..\qemu-img\qemu-img.exe convert -f vmdk -O vhdx -o subformat=dynamic .\hxexsa-disk1.vmdk .\hxexsa-disk1.vhdx

Please use the proper location reference where you installed `qemu-img`.

Here is what the various parameter do:

* `convert` - subcommand to convert images
* `-f` from or source format, here `vmdk`
* `-O` output format, here `vhdx`
* `-o subformat=dynamic` - make it thin-provisioned = don't use up all 138 GB of provisioned space
* source and destination filenames

Depending on your machines I/O performance you will have the converted VHDX file after about 5 minutes time. It grew in size and is now 62 GB proofing that VHDX images are less efficient compared to VMDK.

![Converted VHDX hard disk image](/assets/qemu-img-convert-to-vhdx.png)

[8]: https://en.wikipedia.org/wiki/VHD_(file_format)
[9]: https://en.wikipedia.org/wiki/Fabrice_Bellard

---

[<< Previous Chapter](chapter-2-preparations.md) | [Content Table](README.md) | [Next Chapter >>](chapter-4-create-vm.md)
