---
title: Deploy HANA Express 2.0 on Microsoft Hyper-V - Chapter 4
author: st-gr
date: 5/1/2025
mainfont: Helvetica, Arial, sans-serif
fontsize: 18px
---

SAP HANA Express 2.0 on Hyper-V
===============================

***Guide to deploy HANA Express 2.0 on Microsoft Hyper-V***

**Author:** *st-gr*

[<< Previous Chapter](chapter-3-convert-appliance.md) | [Content Table](README.md) | [Next Chapter >>](chapter-5-post-install.md)

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
| vCPUs            | 4                                        | **4 virtual processors** (leave "Virtual NUMA" enabled)                               |
| Memory           | 16 GB static                             | **32 GB static** (HANA dislikes dynamic memory)                                       |
| System firmware  | BIOS (VMware vmx-09)                     | **Generation 2** VM, Secure Boot = *Microsoft UEFI Certificate Authority*  (SLES 15 is signed; if it won't boot, disable Secure Boot) |
| System disk      | 137 GB stream-optimized VMDK             | Convert to **VHDX (dynamic, 137 GB)** and attach to the SCSI controller               |
| Controllers      | 1 LSI-logic SCSI, 1 IDE                  | Hyper-V adds a SCSI controller automatically; no IDE needed                           |
| NIC              | 1 E1000 on "bridged" network             | 1 synthetic **vNIC** on an **External** virtual switch (bridged)                      |
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
    * Connects the VM's network adapter to the virtual switch called `External`.
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

    blikd

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

