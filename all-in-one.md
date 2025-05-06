---
title: Deploy HANA Express 2.0 on Microsoft Hyper-V
author: st-gr
date: 5/1/2025
mainfont: Helvetica, Arial, sans-serif
fontsize: 18px
---

SAP HANA Express 2.0 on Hyper-V
===============================

---
***Guide to deploy HANA Express 2.0 on Microsoft Hyper-V***

**Author:** *st-gr*

# Chapter One: Motivation
SAP provides a trial version of their in-memory HANA database named HANA Express as an *Open Virtual Appliance (.ova) container*.

Several virtualization software platforms support the Open Virtual Appliance (OVA) format. Prominent examples include VMware Workstation, VirtualBox, and other VMware products like Workstation Player, and vSphere. Additionally, platforms like Red Hat Enterprise Virtualization, Oracle VM, and some cloud services also support OVA.

This list unfortunately does not include Microsoft Hyper-V that is included with the Professional or Enterprise editions of Windows.

This guide will walk you through how to deploy SAP HANA Express or with some modifications ***basically any Linux appliance in the .ova format to Microsoft Hyper-V.***

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

# Chapter Four: Create the VM and attempt to boot

This chapter was created with many trials and errors and includes some advanced Linux concepts like UEFI boot partitions, etc.

First we will copy the converted VHDX file into the folder where Hyper-V expects the virtual disk files. In my setup this is `C:\ProgramData\Microsoft\Windows\Virtual Hard Disks`.

I am going to move the file there to save on disk space.

    Move-Item .\hxexsa-disk1.vhdx "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\"

## The OVF file

The ovf file that we extracted from the OVA file contains the VM configuration. Internally it is a XML file.

We get the following information out of it and need to translate it into the corresponding Hyper-V setting:

| Resource         | Value in the OVF                         | What to set in Hyper-V                                                                 |
|------------------|------------------------------------------|----------------------------------------------------------------------------------------|
| vCPUs            | 4                                        | **4 virtual processors** (leave “Virtual NUMA” enabled)                               |
| Memory           | 16 GB static                             | **32 GB static** (HANA dislikes dynamic memory)                                       |
| System firmware  | BIOS (VMware vmx-09)                     | **Generation 2** VM, Secure Boot = *Microsoft UEFI Certificate Authority*  (SLES 15 is signed; if it won’t boot, disable Secure Boot) |
| System disk      | 137 GB stream-optimized VMDK             | Convert to **VHDX (dynamic, 137 GB)** and attach to the SCSI controller               |
| Controllers      | 1 LSI-logic SCSI, 1 IDE                  | Hyper-V adds a SCSI controller automatically; no IDE needed                           |
| NIC              | 1 E1000 on “bridged” network             | 1 synthetic **vNIC** on an **External** virtual switch (bridged)                      |
| Integration      | VMware Tools                             | Keep default Hyper-V Integration Services (time-sync ON)                              |

I am going to allocate 32 GB of RAM for my application.

## Create a Network Switch

Unlike VMWare Hyper-V can share network switches and therefore they are managed as separate entities.

Use this in an Administrator PowerShell terminal. It will grab the first Network adapter and create and assign the new `External` switch to it:

    New-VMSwitch -Name "External" -NetAdapterName (Get-NetAdapter -Physical | Where Status -eq 'Up' | Select -First 1 -Expand Name) -AllowManagementOS $true

If the assignment was done to the wrong network card then you can change this manually in the `Virtual Switch Manager`

![External Network Switch - your network adapter name will vary](/assets/hyper-v-external-switch.png)

## Create the VM

First we will attempt to create and boot a Gen1 VM that is expecting to boot from a Hard disk on an IDE controller.

You can use the Hyper-V Manager or admin PowerShell:

    New-VM -Name HXEHost -Generation 1 -MemoryStartupBytes 32GB -SwitchName "External"; Set-VMProcessor -VMName HXEHost -Count 4; Add-VMHardDiskDrive -VMName HXEHost -ControllerType IDE -ControllerNumber 0 -ControllerLocation 0 -Path "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\hxexsa-disk1.vhdx"

We are also deactivating Hyper-V checkpoint snapshots and enable shutdown when we opt to stop the appliance:

    Set-VM HXEHost -CheckpointType Disabled
    Set-VM HXEHost -AutomaticStopAction ShutDown

![Create and start Gen1 VM](/assets/hyper-v-create-gen1-vm.png)

### Connect and attempt to boot then Gen1 VM

In the Hyper-V Manager we right-click on the new VM and choose Connect. A window will pop-up that shows the console screen of the appliance.

In this window we click the start button.

We are greeted with the SUSE Grub boot menu and let the boot process continue.

![SLES 15-SP3 grub boot screen](/assets/hyper-v-suse-sles-15-sp3-boot-screen.png)

We get a seemingly endless list of `dracut` timeouts.
It fails again and we land in an emergency shell after 5 minutes.

![SLES 15-SP3 dracut timeouts](/assets/hyper-v-gen1-sles-15-sp3-dracut-timeouts.png)

We can log into the shell using the password `HXEHana1`.

The problem is that no Hyper-V drivers are loaded in the initramfs and no access to the hard drive partitions is possible. 

The dracut command to rebuild the initramfs is not available either:

    dracut --add-drivers "hv_vmbus hv_storvsc" -f reboot -f

![Gen1 VM Boot failure emergency shell](/assets/hyper-v-gen1-emergency-shell.png)

Also attempting to load those drivers in the shell fails as the `hv_storvsc` module is missing.  
This prevents us from fixing the boot issue on-the-fly.

We have some options:

1. Use a compatible OS or rescue disk to chroot into the OS, then configure and rebuild the initramfs.
2. We could also transplant the complete root file system into a compatible and bootable OS image. This option would allow converting the VM to a Gen2 VM, too.

### Option 1: Download compatible Linux Distribution

From the `uname -r` output of the emergency shell and the boot screen we learned that the emergency shell uses the Linux kernel `5.3.18-150300.59.164`. Searching for SUSE and this release number we find that [SLE15 SP3 - LTSS][11] used this kernel. We therefore need to download aa version of *SUSE Enterprise Linux 15 SP3*.

Navigating to [suse.com/download/sles][12] we click on the Stable Release *15 SP3* and Architecture *AMD64/Intel 64*. Searching for *Hyper* we find the file `SLES15-SP3-JeOS.x86_64-15.3-MS-HyperV-GM.vhdx.xz` which already has the VHDX format needed for Hyper-V and is a JeOS version meaning "Just Enough OS" to run.

![Download SUSE SLES 15 SP3 JeOS for MS Hyper-V](/assets/sles-15-sp3-jeos-hyperv.png)

After accepting the terms of use we can extract the VHDX using 7-Zip into the Hyper-V Virtual Hard Drive folder.

![Extract SLES 15-SP3 JeOS into Virtual Hard Drive folder](/assets/7-zip-extract-sles-15-sp3-jeos-for-ms-hyper-v.png)

#### Using JeOS to Install Hyper-V Drivers on Gen1 HANA Express

We will be mounting both VHDX in a temporary VM to fix the boot issue of the appliance.

Enter the below in an admin PowerShell terminal:

    New-VM -Name SLES-15-SP3-JeOS -Generation 1 -MemoryStartupBytes 2GB -SwitchName External -VHDPath "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\SLES15-SP3-JeOS.x86_64-15.3-MS-HyperV-GM.vhdx"; Set-VMProcessor -VMName SLES-15-SP3-JeOS -Count 1; Add-VMHardDiskDrive -VMName SLES-15-SP3-JeOS -ControllerType IDE -ControllerNumber 0 -ControllerLocation 1 -Path "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\hxexsa-disk1.vhdx"

This is what the command does:
1. **New-VM**  
    * Creates a new Generation 1 VM named `SLES-15-SP3-JeOS`
    * Allocates 2 GB of startup RAM to the VM.
    * Connects the VM’s network adapter to the virtual switch called `External`.
    * Points its **primary (boot) disk** at the specified JeOS VHDX file
2. **Set-VMProcessor**
    * Configures the VM with 1 virtual CPUs.
3. **Add-VMHardDiskDrive***
    * Attaches the `hxexsa-disk1.vhdx` file as a secondary IDE disk on controller 0, location 1.

We are also doing the usual snapshot suppression:

    Set-VM SLES-15-SP3-JeOS -CheckpointType Disabled

Let us connect and run the new VM. After the initial setup we should have a shell. I am using Putty to connect to it.

![Booted JeOS with both VHDX connected](/assets/hyper-v-gen1-jeos-booted.png)

Now we mount the second disk (/dev/sdb), chroot into it, enable and bundle the Hyper-V drivers into its initramfs, then clean up.

**1. Get overview of the partitions**

    blkid

Shows

    /dev/sdb1: PARTUUID="4b16cc78-ae46-4a65-b54f-ad5c76f78582"
    
    /dev/sdb2: UUID="0cb08faa-3bbf-4b5f-800c-6ab5de7de4e8" TYPE="swap" PARTUUID="68cc3463-46df-4e3c-b3f9-fcc71916a409"
    
    /dev/sdb3: UUID="768d8aa4-8b83-4660-ac2a-319dbafdb00a" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="c17db041-3247-4d2b-93a5-00f25368aa00"

That means
* sdb1 - boot partition
* sdb2 - swap partition
* sdb3 - root filesystem

**2. Create and mount the root filesystem**
    
    mkdir -p /mnt/sdbroot
    mount /dev/sdb3 /mnt/sdbroot
    
**3. Validate that we mounted the correct root (HANA appliance)**

    ls /mnt/sdbroot
    
Output should show `hana`:

    bin  boot  dev  etc  hana  home  lib  lib64  mnt  opt  proc  root  run  sbin  selinux  srv  sys  tmp  usr  var

Observation: `hana` is present and `boot` is part of the root filesystem and not in a separate partition.

**4. Bind-mount kernel filesystems**

    for fs in dev sys proc run; do mount --bind /$fs /mnt/sdbroot/$fs; done

If the target had a separate /boot then we'd need to mount that, too, e. g.

    # mount /dev/sdb1 /mnt/sdbroot/boot # we don't need this

**5. Chroot and enable Hyper-V support**

````
chroot /mnt/sdbroot /bin/bash -l -c '
  # pick up the kernel version that actually exists under /lib/modules
  kver=$(ls /lib/modules)

  # only enable the daemons you have
  systemctl enable hv_fcopy_daemon.service hv_kvp_daemon.service || true

  # drop in the dracut snippet
  mkdir -p /etc/dracut.conf.d
  printf "add_drivers+=\\"hv_vmbus hv_netvsc hv_storvsc\\"\\n" \
    > /etc/dracut.conf.d/hv-drivers.conf

  # rebuild initramfs for the correct kernel
  dracut --kver "$kver" -f -v
'
````

![Execute rebuild of initramfs](/assets/hyper-v-jeos-fix-hxe-chroot.png)

A bunch of warnings are output, but it should end in:

    dracut: *** Creating initramfs image file '/boot/initrd-5.3.18-150300.59.164-default' done ***
    
**6. Unmount**

    for fs in run proc sys dev; do umount /mnt/sdbroot/$fs; done
    # umount /mnt/sdbroot/boot   # if you mounted it - not needed
    umount /mnt/sdbroot
    
**7. Restart the now fixed HXEhost**

We can now start our VM again and presto the hard drive is detected and the initial boot of the HANA Express Appliance works.

![Changing time zone - HXE booted on Hyper-V](/assets/hyper-v-gen1-vm-hxe-boots.png)

### Option 2: Transplant to use a Gen2 VM

A Hyper-V Gen2 VM boots from UEFI not BIOS and has secure boot enabled by default. A Gen2 VM also has no COM Ports which prevents us from using the COMpipe option below which is more a nice-to-have. A Gen2 VM however can be deployed on Azure.

Since we were able to boot the VHDX using a Gen1 VM we know that the partition scheme of the VHDX is not compatible with an UEFI boot system.

We could try and use a Gparted live CD to add an EFI partition and do other things:

* UUIDs in fstab - Device names (e. g. `/dev/sda*`) can change when we switch from an IDE to SCSI controller use UUIDs not device names in `/etc/fstab`
* Add ESP - Gen 2 firmware only boots from FAT32 EFI partition using a GParted Live CD
* Use VHDX, not VHD - We are prepared as a Gen 2 does not boot VHD disk images
* Shrink the root partition and create a 512 MB EFI System Partition using GParted Live
* grub-install (EFI target) - Places `/EFI/<id>/grub64.efi` and updates NVRAM
* Secure Boot template - Linux shims are signed by MS UEFI CA, not by default Hyper-V template. Might need to disable secure boot.

Microsoft’s official guidance is still “fresh install on Gen 2 and migrate/restore data” .
For complex setups that can be quicker and less risky.

We could use the JeOS as a template and transplant the root filesystem of the HANA Appliance into the JeOS Gen2 VM.
The problem with using the JeOS template is that the root partition of the JeOS uses the Btrfs filesystem whereas the HANA Express Appliance uses the XFS filesystem, see `blkid` output from the **Option 1** paragraphs above.

#### Step 1: Download JeOS

Follow the steps outlined in **Option 1: Download compatible Linux Distribution**.

#### Step 2: Convert the JeOS root partition to XFS

1. First, create copies of the VHDX:
   SHIFT + Right-Click on the folder
   `C:\ProgramData\Microsoft\Windows\Virtual Hard Disks` and open a  terminal (I use PowerShell 5.1).

````
Copy-Item "SLES15-SP3-JeOS.x86_64-15.3-MS-HyperV-GM.vhdx" "HXEhost_Gen2.vhdx"
Copy-Item "SLES15-SP3-JeOS.x86_64-15.3-MS-HyperV-GM.vhdx" "JeOS_rescue.vhdx"
````

2. Elevate the PowerShell session to an Admin shell:

````
Start-Process powershell -Verb RunAs -ArgumentList "-NoExit -Command cd '$PWD'"
````

3. Create a Gen2 VM named "JeOS-Rescue" in the Admin PowerShell:

````
New-VM -Name "JeOS-Rescue" -Generation 2 -MemoryStartupBytes 4GB -VHDPath "JeOS_rescue.vhdx" -SwitchName "External"
Set-VMFirmware -VMName "JeOS-Rescue" -EnableSecureBoot Off
Set-VM JeOS-Rescue -CheckpointType Disabled
Set-VMFirmware -VMName "JeOS-Rescue" -EnableSecureBoot Off
````

4. Boot the rescue VM and log in:
   SUSE JeOS needs a network connection and then starts the jeos-firtsboot to set the initial password of the root user and the locale).

5. Resize the target disk from 24 GB to 140 GB

```
Resize-VHD -Path "HXEhost_Gen2.vhdx" -SizeBytes 140GB
```

6. Add the other VHDX files as additional drives:
   We do this after the VM booted as the UUIDs of the other VHDX are identical to the boot drive. The system could therefore mount partitions on drives that we want to modify.

````
Add-VMHardDiskDrive -VMName "JeOS-Rescue" -Path "HXEhost_Gen2.vhdx"
Add-VMHardDiskDrive -VMName "JeOS-Rescue" -Path "SLES15-SP3-JeOS.x86_64-15.3-MS-HyperV-GM.vhdx"
Add-VMHardDiskDrive -VMName "JeOS-Rescue" -Path "hxexsa-disk1.vhdx"
````

   List all disks to identify them

````
lsblk -o NAME,HCTL,SIZE,FSTYPE,UUID,MOUNTPOINT
````

We observe that 
* /dev/sda = `JeOS_rescue.vhdx`
* /dev/sdb = `HXEhost_Gen2.vhdx` = our target disk
* /dev/sdc = `SLES15-SP3-JeOS.x86_64-15.3-MS-HyperV-GM.vhdx` = original JeOS image
* /dev/sdd = `hxexsa-disk1.vhdx` = converted ova image without EFI partition

![JeOS-Rescue mounts and other drives](/assets/hyper-v-gen2-JeOS-rescue-mountpoints.png)

##### Install xfsprogs package

Since JeOS is a minimal OS we need to manually install the xfsprogs package from a rpm package file.

Use another OS to download the rpm packages from OpenSUSE as the SLES JeOS has no valid package repository without registration.

Fetch the xfsprogs formatter tools:

       wget https://download.opensuse.org/distribution/leap/15.3/repo/oss/x86_64/xfsprogs-4.15.0-4.27.1.x86_64.rpm :contentReference[oaicite:2]{index=2}
       
Upload the file to the JeOS-Rescue VM using WinSCP or SCP from Ubuntu/Linux WSL.

Verify the package requirements

    rpm -qpR xfsprogs-4.15.0-4.27.1.x86_64.rpm

outputs:

````
/bin/bash
/bin/sh
/bin/sh
/bin/sh
/bin/sh
coreutils
libblkid.so.1()(64bit)
libblkid.so.1(BLKID_2.15)(64bit)
libblkid.so.1(BLKID_2.17)(64bit)
libc.so.6()(64bit)
libc.so.6(GLIBC_2.10)(64bit)
libc.so.6(GLIBC_2.14)(64bit)
libc.so.6(GLIBC_2.2.5)(64bit)
libc.so.6(GLIBC_2.26)(64bit)
libc.so.6(GLIBC_2.3)(64bit)
libc.so.6(GLIBC_2.3.3)(64bit)
libc.so.6(GLIBC_2.3.4)(64bit)
libc.so.6(GLIBC_2.4)(64bit)
libc.so.6(GLIBC_2.6)(64bit)
libhandle.so.1()(64bit)
libhandle.so.1(LIBHANDLE_1.0.3)(64bit)
libpthread.so.0()(64bit)
libpthread.so.0(GLIBC_2.2.5)(64bit)
libpthread.so.0(GLIBC_2.3.2)(64bit)
libreadline.so.7()(64bit)
librt.so.1()(64bit)
librt.so.1(GLIBC_2.2.5)(64bit)
librt.so.1(GLIBC_2.3.3)(64bit)
libuuid.so.1()(64bit)
libuuid.so.1(UUID_1.0)(64bit)
rpmlib(CompressedFileNames) <= 3.0.4-1
rpmlib(FileDigests) <= 4.6.0-1
rpmlib(PayloadFilesHavePrefix) <= 4.0-1
rpmlib(PayloadIsXz) <= 5.2-1
````

This boils down to the following four additional packages:

* coreutils 
* util-linux
* libhandle1
* libreadline7

Let us check which ones are already installed:

    rpm -q coreutils util-linux libhandle1 libreadline7

outputs:

````
coreutils-8.32-1.2.x86_64
util-linux-2.36.2-2.29.x86_64
package libhandle1 is not installed
libreadline7-7.0-17.83.x86_64
````

It is a little misleading as libhandle1 is part of the xfsprogs package. So no additional package is needed.

Install the XFS tools

    rpm -Uvh xfsprogs-4.15.0-4.27.1.x86_64.rpm
    
Confirm that `mkfs.xfs` works

    mkfs.xfs -V
    # output
    mkfs.xfs version 4.15.0

##### Recreate the root partition as XFS

First, let's mount the target disk (HXEHost_Gen2.vhdx, which is sdb, but we need to change its UUID as it is already in use)

    mkdir -p /mnt/target
    NEW_UUID=$(uuidgen)
    btrfstune -U $NEW_UUID /dev/sdb3   # confirm warning with y
    mount /dev/sdb3 /mnt/target
    
Check that we can access the filesystem

    ls /mnt/target
    
Create new XFS filesystem this will delete the partition!

    umount /mnt/target
    mkfs.xfs -f /dev/sdb3

Resize /dev/sdb3 to 148 of 150GB of the disk

````
parted /dev/sdb resizepart 3 148GB
partprobe /dev/sdb
````

Grow the XFS

    mount /dev/sdb3 /mnt/target
    xfs_growfs /mnt/target
    umount /mnt/target

##### Copy the appliance root

````
# mount source and target
mkdir -p /mnt/source
mount -o ro /dev/sdd3 /mnt/source
mount /dev/sdb3    /mnt/target

# copy *everything* EXCEPT the ESP and runtime dirs
rsync -aAXH --info=progress2 \
      --exclude='/boot/efi/*' \
      --exclude='/proc/*' --exclude='/sys/*' --exclude='/dev/*' \
      --exclude='/run/*'  --exclude='/tmp/*' \
      /mnt/source/  /mnt/target/
````

#### Step 3: Configure EFI, Swap partitions, fstab

##### Create a 2 GiB swap partition on sdb

````
umount /mnt/target
parted -s /dev/sdb mkpart primary linux-swap 148GB 100%
partprobe /dev/sdb
mkswap /dev/sdb4
uuid_swap=$(blkid -s UUID -o value /dev/sdb4)
````

##### Give sdb2 a unique ESP UUID & mount it

````
# re‑format the ESP to get a fresh UUID
mkfs.vfat -F32 -n EFI /dev/sdb2
uuid_esp=$(blkid -s UUID -o value /dev/sdb2)

# mount it under the new root
mkdir -p /mnt/target/boot/efi
mount /dev/sdb2 /mnt/target/boot/efi
````

(*We leave the tiny BIOS‑GRUB partition **sdb1** untouched.*)

##### Bind pseudo‑fs and chroot into the new system

````
for d in dev proc sys run; do mount --bind /$d /mnt/target/$d; done
chroot /mnt/target /bin/bash
````

##### Create a clean /etc/fstab

````
uuid_root=$(blkid -s UUID -o value /dev/sdb3)
cat > /etc/fstab <<EOF
UUID=$uuid_root  /          xfs    defaults              0 1
UUID=$uuid_esp   /boot/efi  vfat   umask=0002,utf8      0 2
UUID=$uuid_swap  swap       swap   defaults              0 0
EOF
````

Validate that all three partitions are referenced with a UUID.

    cat /etc/fstab
    
If not all partitions have a UUID, then manually determine them with `lsblkid -f` and then edit fstab with `vi /etc/fstab`.

##### Rebuild initrd with Hyper‑V drivers

````
kv=$(ls /boot/vmlinuz-* | head -1 | sed 's#.*/vmlinuz-##')
dracut --add-drivers "hv_vmbus hv_storvsc" -f "/boot/initrd-$kv" "$kv"
````

##### Install GRUB to the new ESP & regenerate menu

````
exit # get out of chroot

# create and mount the ESP inside that root
mkdir -p /mnt/target/boot/efi
mount /dev/sdb2   /mnt/target/boot/efi

# quick sanity check
ls /mnt/target/boot/efi/EFI 2>/dev/null || echo "ESP is empty (that's OK for now)"

grub2-install --target=x86_64-efi \
             --boot-directory=/mnt/target/boot \
             --efi-directory=/mnt/target/boot/efi \
             --bootloader-id=SLES \
             --removable

# Finish inside the chroot - create the GRUB menu
chroot /mnt/target /bin/bash
grub2-mkconfig -o /boot/grub2/grub.cfg

exit # get out of chroot
````

##### Clean up & reboot

````
umount -R /mnt/target
poweroff        # shut down, then detach the helper disks
````

#### Step 4: Create and boot the Gen2 VM

````
New-VM -Name "HXEHost_Gen2" -Generation 2 -MemoryStartupBytes 32GB -VHDPath "C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\HXEHost_Gen2.vhdx" -SwitchName "External"
    
Set-VMProcessor -VMName "HXEHost_Gen2" -Count 4
Set-VMFirmware -VMName "HXEHost_Gen2" -EnableSecureBoot Off
Set-VM -Name "HXEHost_Gen2" -CheckpointType Disabled
````

This will:
1. Create a Gen2 VM with 32GB RAM and the specified VHDX
2. Set it to use 4 virtual CPUs
3. Disable Secure Boot
4. Disable checkpoints

### Optional: Use COMpipe to connect to the VM via PuTTY

At boot time the Hyper-V VM has no network connection and no OpenSSH service running. Therefore we can't use PuTTY or another terminal tool to interact with the VM at this stage.

PuTTY can also use a COM port to connect to a machine and Hyper-V lets us bind the VM's COM ports to named pipes. PuTTY can't directly use named pipes. That is where COMpipe comes into play. It bridges the host machines (your PC) physical COM port with a named pipe then PuTTY can connect to this tunnel to the VM's shell.

Make sure that you have an available COM port otherwise use a virtual COM port tool like [com0com][10].

**1. Configure a named pipe name on the VM's COM1 port**  
    Let us name it `hxe2com4`.
  
![Configure named pipe for guest VM COM1](/assets/hyper-v-com1-pipe.png)
    
**2. Start the VM and pause boot and enable serial console**  
    Right click on the VM in Hyper-V Manager and chose Connect, then click the start button.
    When you are presented with a boot menu press the cursor down key to toggle the grub menu selection. This will interrupt the timeout for the default boot item.
    Enable the serial console, by pressing the 'e' key and then move the cursor until you see the depicted boot entry. Replace `splash=silent quiet` with `console=ttyS0,9600`. Don't press F10, yet only after step 4.
    
![Edit grub boot entry, enable serial console](/assets/hyper-v-gen1-sles-grub-enable-serial-console.png)

**3. Start COMpipe from an admin PowerShell terminal**  
    This will tunnel the VM's COM1 to the hosts COM4 port.

    COMpipe.exe -b 9600 -c \\.\COM4 -p \\.\pipe\hxe2com4

**4. Use PuTTY to connect**  
    Connect to COM4 (when using com0com, then COM6) in PuTTY.
    
**5. Continue booting**  
    Press F10 to continue to boot. PuTTY will now show the console output.

![COMpipe running](/assets/hyper-v-gen1-sles-putty-serial-console-access.png)

As a result we can now use PuTTY to interact with the VM even if it has no LAN connectivity.

![PuTTY over COM port serial console](/assets/hyper-v-gen1-putty-over-serial-console.png)


[10]: https://sourceforge.net/projects/com0com/

[11]: https://www.suse.com/support/kb/doc/?id=000019587#:~:text=5.3.18%2D150300.59.164.1

[12]: https://www.suse.com/download/sles/

---

# Chapter Five: HANA Express post-install tasks

## Maintain hosts file

With our appliance SAP delivered a `Getting_Started_HANAexpress_VM.pdf` file. There we learn to update our hosts file so that `hxehost.localdomain` can be resolved. This also requires a static IP.

In an admin PowerShell terminal type:

    start notepad C:\Windows\System32\drivers\etc\hosts
    
Add the entry as described and save. The IP of the Appliance is shown in the Hyper-V Virtual Machine Connection window.

## First logon - set root and HANA DB system passwords

Upon first logon we must replace the default `hxeadm` users password `HXEHana1` with a new one.

We also must set the HANA system users password.

![First logon to the Appliance](/assets/hana-express-first-logon.png)

## Initial start and configuration

It takes about 18 to 20 minutes until the HANA Express DB is  configured and ready for transactions.

![Initial start](/assets/hdb-initial-config-18-20-min.png)

You can test that the HDB is functioning with a simple select:

    hdbsql -i 90 -d SystemDB -u SYSTEM "SELECT CURRENT_TIMESTAMP FROM DUMMY;"
    
Enter the chosen SYSTEM users password and then you should see the current UTC timestamp.

## Open firewall ports

The command `xs apps` lists all HANA applications with their ports. Per default none of the ports are reachable. We must allow traffic to pass:

````
sudo firewall-cmd --zone=public --add-port=30015/tcp --permanent
sudo firewall-cmd --zone=public --add-port=30013/tcp --permanent
sudo firewall-cmd --zone=public --add-port=30017/tcp --permanent
sudo firewall-cmd --zone=public --add-port=39015/tcp --permanent
sudo firewall-cmd --zone=public --add-port=59013/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8090/tcp --permanent
sudo firewall-cmd --zone=public --add-port=53075/tcp --permanent
sudo firewall-cmd --zone=public --add-port=39032/tcp --permanent
sudo firewall-cmd --zone=public --add-port=51035/tcp --permanent

sudo firewall-cmd --reload
````

## Starting and stopping the HDB

To start the database use

    HDB start
    
and wait 18 to 20 minutes for the services to become available.

To stop the database use

    HDB stop
    
Wait for the `hdbdaemon is stopped.` output. Then you can `sudo poweroff` the virtual machine.

## License

This documentation is licensed under a [Creative Commons Attribution 4.0 International License](LICENSE).

---