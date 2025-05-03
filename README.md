# SAP HANA Express 2.0 on Hyper-V

![SAP HANA on Hyper-V](/assets/cover.png)

***Complete Guide to deploy HANA Express 2.0 on Microsoft Hyper-V***

**Author:** *st-gr*

## Introduction

SAP provides a trial version of their in-memory HANA database named HANA Express as an *Open Virtual Appliance (.ova) container*. While several virtualization platforms support the OVA format, Microsoft Hyper-V (included with Professional or Enterprise editions of Windows) is not one of them.

This guide will walk you through how to deploy SAP HANA Express or with some modifications ***basically any Linux appliance in the .ova format to Microsoft Hyper-V.***

## Contents

1. [Motivation](chapter-1-motivation.md)
2. [Preparations](chapter-2-preparations.md)
3. [Convert the virtual appliance](chapter-3-convert-appliance.md)
4. [Create the VM and attempt to boot](chapter-4-create-vm.md)
5. [HANA Express post-install tasks](chapter-5-post-install.md)

---

## Quick Start

If you're already familiar with virtualization concepts, here's a quick overview of the process:

1. Download SAP HANA Express from the [SAP HANA trial page](https://www.sap.com/products/data-cloud/hana/express-trial.html)
2. Extract the .ova file and convert the VMDK to VHDX format
3. Create a Gen1 Hyper-V virtual machine with the converted disk
4. Fix boot issues using SLES 15 SP3 JeOS
5. Complete the HANA Express setup and configuration

For detailed instructions, follow the chapters in order.

---

## License

This work is licensed under a [Creative Commons Attribution 4.0 International License](LICENSE).

*Last updated: 5/1/2025*