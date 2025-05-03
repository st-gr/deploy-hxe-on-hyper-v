---
title: Deploy HANA Express 2.0 on Microsoft Hyper-V - Chapter 5
author: st-gr
date: 5/1/2025
mainfont: Helvetica, Arial, sans-serif
fontsize: 18px
---

SAP HANA Express 2.0 on Hyper-V
===============================

***Guide to deploy HANA Express 2.0 on Microsoft Hyper-V***

**Author:** *st-gr*

[<< Previous Chapter](chapter-4-create-vm.md) | [Content Table](README.md)

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

