---
published: true
date: '2025-11-04 17:58 +0100'
title: Cisco IOS XRv 9000 UCS M7 Appliance software upgrade
author: Frederic Cuiller
excerpt: >-
  This document aims to describe and document how to upgrade Cisco IOS XR
  software running on a Cisco IOS XRv 9000 UCS M7 appliance.
tags:
  - iosxr
  - Appliance
  - RR
  - XRv 9000
position: top
---
{% include toc icon="table" title="Cisco IOS XRv 9000 UCS M7 appliance software upgrade" %}

# Introduction

This document aims to describe and document how to upgrade Cisco IOS XR software running on a Cisco IOS XRv 9000 UCS M7 appliance.

There are two main cases to consider:

- Software reimage: this technique is usually used for initial staging or disaster recovery. The procedure is documented in details in this blog post
- Traditional software upgrade: the appliance is already running a certain IOS XR version which must be upgraded. This second case is documented in [this blog post]({{site.baseurl}}/tutorials/xrv-9000-ucs-m7-appliance-reimage).

This article will strictly focus on the software upgrade aspect, it does not cover:

- Software upgrade pre-checks and post-checks
- BGP RR isolation and draining best practices

# Prerequisite

The files required to perform software upgrade can be obtained on cisco.com in the [IOS XRv 9000 Download section](https://software.cisco.com/download/home/286288939/type/280805694/release).  

![xrv9k-download-section.png]({{site.baseurl}}/images/xrv9k-download-section.png)

Always download and read the software upgrade document, as it contains important information about potential mandatory bridge SMU and rollback procedure.
The individual mini ISO, optional RPMs packages, and a pre-built golden ISO file can be downloaded from the non VRR profile TAR archive: once the appliance is baked and flagged with the VRR profile, any ISO file can be used to perform IOS XR software upgrade.

Content of the archive is the following:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
{25-11-04 17:16}FCUILLER-M-NQD1:~/Downloads/ISO/fullk9-R-XRV9000-2522 fcuiller% ls -l
total 18512656
-rw-------@ 1 fcuiller  staff        2320 Sep 22 20:48 README-fullk9-R-XRV9000-2522.txt
-r-xr-xr-x@ 1 fcuiller  staff     5656121 Sep 21 00:27 xrv9k-bng-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff      261855 Sep 21 00:26 xrv9k-bng-supp-x64-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff      641424 Sep 21 00:26 xrv9k-eigrp-1.0.0.0-r2522.x86_64.rpm
-rwxr-x---@ 1 fcuiller  staff  1733353472 Sep 21 00:49 xrv9k-fullk9-x-25.2.2.iso
-rwxr-x---@ 1 fcuiller  staff  2101237760 Sep 21 00:53 xrv9k-fullk9-x-25.2.2.ova
-rwxr-x---@ 1 fcuiller  staff  2150957056 Sep 21 00:55 xrv9k-fullk9-x-25.2.2.qcow2
-rw-r--r--@ 1 fcuiller  staff  1734103040 Sep 21 01:48 xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2.iso
-r-xr-xr-x@ 1 fcuiller  staff     4073292 Sep 21 00:27 xrv9k-isis-1.0.0.0-r2522.x86_64.rpm
-rwxr-x---@ 1 fcuiller  staff        9248 Sep 21 00:24 xrv9k-k9sec-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff      367774 Sep 21 00:26 xrv9k-li-x-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff        9219 Sep 21 00:24 xrv9k-m2m-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff    12080898 Sep 21 00:27 xrv9k-mcast-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff     1940981 Sep 21 00:26 xrv9k-mgbl-1.0.0.0-r2522.x86_64.rpm
-rw-r--r--@ 1 fcuiller  staff  1694337024 Sep 21 00:32 xrv9k-mini-x-25.2.2.iso
-r-xr-xr-x@ 1 fcuiller  staff     2329691 Sep 21 00:27 xrv9k-mpls-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff     8500488 Sep 21 00:27 xrv9k-mpls-te-rsvp-1.0.0.0-r2522.x86_64.rpm
-r-xr-xr-x@ 1 fcuiller  staff     4116598 Sep 21 00:27 xrv9k-ospf-1.0.0.0-r2522.x86_64.rpm
</code>
</pre>
</div>

# IOS XR Software Upgrade

This section will be short: IOS XR software upgrades are consistent between IOS XRv 9000 UCS M7 appliance and any ASR 9000 running IOS XR 64bit software.
Users have the choice between 2 techniques:

- Pickup the mini ISO file from a given release, optional packages, SMUs, and use the legendary and traditional install add/activate/commit commands, specifying individual files as argument
- Build a custom golden ISO or use the Cisco provided golden ISO, and use the simpler and faster install replace command. This is the recommended way: not only it makes software upgrades simpler, but the process and the same golden ISO file can also be used to patch (install SMU) an existing system. This technique is illustrated below.

After copying golden ISO on the appliance harddisk, and when BGP RR is in maintenance mode, user can start the upgrade process using <code>install replace</code> command:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:IOS-XRv9000_M7_Appliance#<mark>install replace /harddisk:/xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2.iso synchronous</mark>
Tue Nov  4 16:27:25.284 UTC
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
2025-11-04 16:27:32 Install operation 1 started by cisco:
2025-11-04 16:27:32   install replace source /harddisk:/xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2.iso synchronous
0/RP0/ADMIN0:Nov  4 16:27:32.000 UTC: cal_logger[61539]: %OS-SYSLOG-6-LOG_INFO : truncated by file_cleanup.py, it is a cleanup mechanism and will not impact any functionality
2025-11-04 16:27:37 No install operation in progress at this moment
2025-11-04 16:27:37 Checking system is ready for install operation
2025-11-04 16:27:37 'install replace' in progress
2025-11-04 16:27:37 Label = PROD_BUILD_25_2_2
2025-11-04 16:27:37 ISO xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2.iso in input package list. Going to upgrade the system to version 25.2.2.
2025-11-04 16:27:40 Scheme : localdisk
2025-11-04 16:27:40 Hostname : localhost
2025-11-04 16:27:40 Username : None
2025-11-04 16:27:40 SourceDir : /harddisk:
2025-11-04 16:27:40 Collecting software state..
2025-11-04 16:27:40 Getting platform
2025-11-04 16:27:40 Getting supported architecture
2025-11-04 16:27:40 Getting active packages from XR
2025-11-04 16:27:41 Getting inactive packages from XR
2025-11-04 16:27:48 Getting list of RPMs in local repo
2025-11-04 16:27:48 Getting list of provides of all active packages
2025-11-04 16:27:48 Getting provides of each rpm in repo
2025-11-04 16:27:48 Getting requires of each rpm in repo
2025-11-04 16:27:48 Fetching .... xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2.iso
RP/0/RP0/CPU0:Nov  4 16:27:55.700 UTC: sdr_instmgr[1291]: %PKT_INFRA-FM-6-FAULT_INFO : INSTALL-IN-PROGRESS :DECLARE :0/RP0/CPU0: INSTALL_IN_PROGRESS Alarm : being DECLARED for the system
0/RP0/ADMIN0:Nov  4 16:27:55.000 UTC: inst_mgr[3364]: %PKT_INFRA-FM-6-FAULT_INFO : INSTALL-IN-PROGRESS :DECLARE :0/RP0: Sysadmin INSTALL_IN_PROGRESS Alarm : being DECLARED for the system
2025-11-04 16:27:55 Adding packages
	xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2.iso
2025-11-04 16:27:55 Allocated operation ID 2 for install add
2025-11-04 16:28:14 Activating xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2
This install operation will reload the system, continue?
 [yes:no]:[yes]
yes
2025-11-04 16:28:26 Optimized list to prepare after sanitizing input list for superseded packages:
	xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2
2025-11-04 16:28:26 Package list:
2025-11-04 16:28:26 	xrv9k-goldenk9-x-25.2.2-PROD_BUILD_25_2_2
2025-11-04 16:28:26 The operation is past point-of-no-return and can no longer be stopped!
2025-11-04 16:28:26 Action 1: install prepare action started
2025-11-04 16:28:26 Triggering prepare operation.
This may take a while...
^@2025-11-04 16:30:30 Action 1: install prepare action completed successfully
2025-11-04 16:30:32 Prepare operation completed. Trigger activate.
This may take a while...
2025-11-04 16:30:32 Prepare completed. Operation ID 1 will be taken up for activate operation
2025-11-04 16:30:34 Activate operation ID is: 1 for 'install source' ID:1
2025-11-04 16:30:36 Install operation 1 started:
  install activate noprompt synchronous
2025-11-04 16:30:37 Action 1: install activate action started
2025-11-04 16:30:37 The software will be activated with reload upgrade
2025-11-04 16:30:39 Following nodes are available for System Upgrade activate:
2025-11-04 16:30:39  0/RP0
^@^@2025-11-04 16:32:49 Action 1: install activate action completed successfully
RP/0/RP0/CPU0:Nov  4 16:32:58.916 UTC: sdr_instmgr[1291]: %MGBL-SCONBKUP-6-INTERNAL_INFO : Reload debug script successfully spawned
2025-11-04 16:33:03 Install operation 1 finished successfully
2025-11-04 16:33:03 Ending operation 1
RP/0/RP0/CPU0:Nov  4 16:33:03.089 UTC: sdr_instmgr[1291]: %INSTALL-INSTMGR-2-OPERATION_SUCCESS : Install operation 1 finished successfully
RP/0/RP0/CPU0:Nov  4 16:33:03.633 UTC: sdr_instmgr[1291]: %INSTALL-INSTMGR-2-SYSTEM_RELOAD_INFO : The whole system will be reloaded to complete install operation 1
0/RP0/ADMIN0:Nov  4 16:33:05.000 UTC: inst_mgr[3364]: %INFRA-INSTMGR-5-OPERATION_TO_RELOAD : This rack will now reload as part of the install operation
^@Connection to 172.20.163.15 closed by remote host.
Connection to 172.20.163.15 closed.
</code>
</pre>
</div>

The appliance reloads on the new version of code. The whole process takes approximatively 8min.  

Once appliance has rebooted, and post checks are conform, new version can be committed:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:IOS-XRv9000_M7_Appliance#<mark>sh install active summary</mark>
Tue Nov  4 16:42:01.041 UTC
<mark>Label : 25.2.2-PROD_BUILD_25_2_2</mark>

    Active Packages: 13
        xrv9k-xr-25.2.2 version=25.2.2 [Boot image]
        xrv9k-k9sec-1.0.0.0-r2522
        xrv9k-m2m-1.0.0.0-r2522
        xrv9k-eigrp-1.0.0.0-r2522
        xrv9k-bng-supp-x64-1.0.0.0-r2522
        xrv9k-mgbl-1.0.0.0-r2522
        xrv9k-li-x-1.0.0.0-r2522
        xrv9k-bng-1.0.0.0-r2522
        xrv9k-isis-1.0.0.0-r2522
        xrv9k-mcast-1.0.0.0-r2522
        xrv9k-ospf-1.0.0.0-r2522
        xrv9k-mpls-1.0.0.0-r2522
        xrv9k-mpls-te-rsvp-1.0.0.0-r2522

RP/0/RP0/CPU0:IOS-XRv9000_M7_Appliance#<mark>sh install committed summary</mark>
Tue Nov  4 16:42:19.678 UTC
<mark>Label : 24.4.2</mark>

    Committed Packages: 1
        xrv9k-xr-24.4.2 version=24.4.2 [Boot image]

RP/0/RP0/CPU0:IOS-XRv9000_M7_Appliance#<mark>install commit synchronous</mark>
Tue Nov  4 16:42:29.630 UTC
2025-11-04 16:42:33 Install operation 3 started by cisco:
  install commit synchronous
2025-11-04 16:43:00 Install operation 3 finished successfully
2025-11-04 16:43:00 Ending operation 3
RP/0/RP0/CPU0:IOS-XRv9000_M7_Appliance#<mark>sh install committed summary</mark>
Tue Nov  4 16:43:11.657 UTC
<mark>Label : 25.2.2-PROD_BUILD_25_2_2</mark>

    Committed Packages: 13
        xrv9k-xr-25.2.2 version=25.2.2 [Boot image]
        xrv9k-k9sec-1.0.0.0-r2522
        xrv9k-m2m-1.0.0.0-r2522
        xrv9k-eigrp-1.0.0.0-r2522
        xrv9k-bng-supp-x64-1.0.0.0-r2522
        xrv9k-mgbl-1.0.0.0-r2522
        xrv9k-li-x-1.0.0.0-r2522
        xrv9k-bng-1.0.0.0-r2522
        xrv9k-isis-1.0.0.0-r2522
        xrv9k-mcast-1.0.0.0-r2522
        xrv9k-ospf-1.0.0.0-r2522
        xrv9k-mpls-1.0.0.0-r2522
        xrv9k-mpls-te-rsvp-1.0.0.0-r2522

RP/0/RP0/CPU0:IOS-XRv9000_M7_Appliance#
</code>
</pre>
</div>

# Appliance Firmware Management

UCS CIMC, BIOS, NIC drivers, RAID controller firmware etc. can be considered like Field Programmable Devices (FPD).  Like any software, those firmwares might need software upgrade to add new functionality, fix bugs, or improve security.  

While ASR 9000 contains a specific FPD package to cover those components, this is currently not the case on the appliance. Instead, those components software upgrade is currently handled at CIMC level. Always use the supported combination, which is documented in the release notes, and always refer to the latest [documentation](https://www.cisco.com/c/en/us/td/docs/routers/virtual-routers/configuration/guide/b-xrv9k-cg/b-xrv9k-cg_chapter_01010.html#firmware-management):

![xrv9k-firmware-combination.png]({{site.baseurl}}/images/xrv9k-firmware-combination.png)


**Important** Do not download CIMC/BIOS files directly from UCS software download section. Instead, download the signed images from the IOS XRv 9000 download section.
{: .notice--danger}

![xrv9k-firmware-download.png]({{site.baseurl}}/images/xrv9k-firmware-download.png)

The IOS XRv 9000 appliance Host Upgrade Utility (HUU) TAR archive contains the files firmware files required for this operation:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
{25-11-04 15:51}FCUILLER-M-NQD1:~/Downloads/ISO/xrv9k-ucs-c220m7-huu-container-4.3.5.250001/firmware fcuiller% ls -l
total 0
drwxr-xr-x@   3 fcuiller  staff    96 Mar 25  2025 bios
drwxr-xr-x@   4 fcuiller  staff   128 Oct 21 11:38 cimc
drwxr-xr-x@ 136 fcuiller  staff  4352 Mar 25  2025 Common
drwxr-xr-x@   3 fcuiller  staff    96 Mar 25  2025 MOUNTRAINIER1
drwxr-xr-x@   4 fcuiller  staff   128 Mar 25  2025 VIC_FIRMWARE
</code>
</pre>
</div>

- CIMC firmware is located in <code>cimc/cimc.bin</code>
- Signed BIOS is located in <code>bios/bios.pkg</code>

The next part of the article will describe two ways to upgrade firmwares:
- CIMC GUI
- CIMC CLI

## CIMC GUI

Login on the appliance CIMC, navigate to the Administration plane, and select Firmware Management. Select CIMC to upgrade the CIMC:
![xrv9k-cimc-management.png]({{site.baseurl}}/images/xrv9k-cimc-management.png)

Upload the file:
![xrv9k-cimc-upload.png]({{site.baseurl}}/images/xrv9k-cimc-upload.png)


Once it’s uploaded, file integrity is verified and new firmware can be activated. Click on Activate:
![xrv9k-cimc-activation.png]({{site.baseurl}}/images/xrv9k-cimc-activation.png)

CIMC upgrade kicks in. During activation, CIMC access is temporarily unavailable. Operation takes ~120s. CIMC upgrade does not impact underlying UCS workload, which in this case is the IOS XRv 9000 route-reflector.

After activation and CIMC reconnection, the new version is running:
![xrv9k-cimc-new-version-1.png]({{site.baseurl}}/images/xrv9k-cimc-new-version-1.png)

![xrv9k-cimc-new-version-2.png]({{site.baseurl}}/images/xrv9k-cimc-new-version-2.png)

For BIOS upgrade, same process is used: the BIOS file must be uploaded, and activated.
![bios-upload.png]({{site.baseurl}}/images/bios-upload.png)

Unlike CIMC, BIOS activation requires host reboot to be effective, meaning impact on underlying IOS XRv 9000 software running on the appliance:
![bios-requires-reboot.png]({{site.baseurl}}/images/bios-requires-reboot.png)

Appliance can be rebooted:
![bios-reboot.png]({{site.baseurl}}/images/bios-reboot.png)


On the KVM, the new version can be already observed during boot process:
![bios-kvm.png]({{site.baseurl}}/images/bios-kvm.png)

And finally new version is visible in CIMC. After this, both CIMC and BIOS versions are the recommended versions for this release:
![target-combination.png]({{site.baseurl}}/images/target-combination.png)


## CIMC CLI

Getting access to CIMC GUI can often be challenging in out-of-band administration networks. It’s also possible to execute this procedure using the CIMC CLI.  

User must log in CIMC via SSH. From here, current CIMC version can be verified:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C# scope cimc
C220-WZP28359K2C /cimc # show firmware
Update Stage Update Progress <mark>Current FW Version</mark>
------------ --------------- ------------------
NONE         0               <mark>4.3(5.240021)</mark>
</code>
</pre>
</div>

CIMC firmware can be uploaded via multiple protocols, and the progress can be monitored:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C /cimc/firmware # update http 172.20.166.52 /cimc.bin
Firmware update initialized.
Please check the status using "show detail".
C220-WZP28359K2C /cimc/firmware # show detail
Firmware Image Information:
    <mark>Update Stage: DOWNLOAD</mark>
    <mark>Update Progress: 5</mark>
    Current FW Version: 4.3(5.240021)
    FW Image 1 Version: 4.3(5.240021)
    FW Image 1 State: BACKUP INACTIVATED
    FW Image 2 Version: 4.3(5.240021)
    FW Image 2 State: RUNNING ACTIVATED
    Boot-loader Version: 4.3(5.240021)
    Secure Boot: DISABLED
C220-WZP28359K2C /cimc/firmware # show detail
Firmware Image Information:
    <mark>Update Stage: INSTALL</mark>
    <mark>Update Progress: 45</mark>
    Current FW Version: 4.3(5.240021)
    FW Image 1 Version: 4.3(5.240021)
    FW Image 1 State: BACKUP INACTIVATED
    FW Image 2 Version: 4.3(5.240021)
    FW Image 2 State: RUNNING ACTIVATED
    Boot-loader Version: 4.3(5.240021)
    Secure Boot: DISABLED
C220-WZP28359K2C /cimc/firmware # show detail
Firmware Image Information:
    <mark>Update Stage: NONE</mark>
    <mark>Update Progress: 100</mark>
    Current FW Version: 4.3(5.240021)
    FW Image 1 Version: 4.3(5.250001)
    FW Image 1 State: BACKUP INACTIVATED
    FW Image 2 Version: 4.3(5.240021)
    FW Image 2 State: RUNNING ACTIVATED
    Boot-loader Version: 4.3(5.240021)
    Secure Boot: DISABLED
C220-WZP28359K2C /cimc/firmware #
</code>
</pre>
</div>

After CIMC firmware upload, new version can be activated and CIMC access is temporarily unavailable:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C /cimc/firmware # <mark>activate</mark>
This operation will activate firmware 1 and reboot the BMC.
Continue?[y|N]<mark>y</mark>
--- connection to CIMC is lost and must be reestablished
C220-WZP28359K2C /cimc/firmware # Read from remote host 172.20.166.102: Operation timed out
Connection to 172.20.166.102 closed.
client_loop: send disconnect: Broken pipe
</code>
</pre>
</div>

After logging back to CIMC CLI, new version can be seen as active:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C# scope cimc
C220-WZP28359K2C /cimc # scope firmware
sC220-WZP28359K2C /cimc/firmware # show detail
Firmware Image Information:
    Update Stage: NONE
    Update Progress: 0
    <mark>Current FW Version: 4.3(5.250001)</mark>
    FW Image 1 Version: 4.3(5.250001)
    FW Image 1 State: RUNNING ACTIVATED
    FW Image 2 Version: 4.3(5.240021)
    FW Image 2 State: BACKUP INACTIVATED
    Boot-loader Version: 4.3(5.250001)
    Secure Boot: DISABLED
</code>
</pre>
</div>

For BIOS upgrade, exact same procedure exists and happens inside a different CIMC scope:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C# <mark>scope bios</mark>
C220-WZP28359K2C /bios # show detail
BIOS:
    <mark>BIOS Version: C220M7.4.3.5a.0_XRV9K</mark>
    Backup BIOS Version: C220M7.4.3.5a.0_XRV9K
    Boot Order: CDROM,HDD
    FW Update Status: None, OK
    UEFI Secure Boot: enabled
    Actual Boot Mode: Uefi
    Last Configured Boot Order Source: CIMC
    One time boot device: (none)
</code>
</pre>
</div>

The BIOS file is uploaded, signature verified:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C /bios # <mark>update http 172.20.166.52 /bios.pkg</mark>
bios update has started.
Please check the status using "show detail".
C220-WZP28359K2C /bios # show detail
BIOS:
    BIOS Version: C220M7.4.3.5a.0_XRV9K
    Backup BIOS Version: C220M7.4.3.5a.0_XRV9K
    Boot Order: CDROM,HDD
    <mark>FW Update Status: Image Download (0 %), OK</mark>
    UEFI Secure Boot: enabled
    Actual Boot Mode: Uefi
    Last Configured Boot Order Source: CIMC
    One time boot device: (none)
C220-WZP28359K2C /bios # show detail
BIOS:
    BIOS Version: C220M7.4.3.5a.0_XRV9K
    Backup BIOS Version: C220M7.4.3.5a.0_XRV9K
    Boot Order: CDROM,HDD
    <mark>FW Update Status: Write Host Flash (50 %), OK</mark>
    UEFI Secure Boot: enabled
    Actual Boot Mode: Uefi
    Last Configured Boot Order Source: CIMC
    One time boot device: (none)
C220-WZP28359K2C /bios # show detail
BIOS:
    BIOS Version: C220M7.4.3.5a.0_XRV9K
    Backup BIOS Version: C220M7.4.3.5b.0_XRV9K
    Boot Order: CDROM,HDD
    <mark>FW Update Status: Done, OK</mark>
    UEFI Secure Boot: enabled
    Actual Boot Mode: Uefi
    Last Configured Boot Order Source: CIMC
    One time boot device: (none)
C220-WZP28359K2C /bios #
</code>
</pre>
</div>

And then it can be activated, after an appliance reload:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
220-WZP28359K2C /bios # <mark>activate</mark>
System is powered-on. This operation will activate backup BIOS version "C220M7.4.3.5b.0_XRV9K" during next boot.
Continue?[y|N]y
C220-WZP28359K2C /bios # exit
C220-WZP28359K2C# <mark>scope chassis</mark>
C220-WZP28359K2C /chassis # <mark>power cycle</mark>
</code>
</pre>
</div>

After reboot and logging in CIMC, new version can be observed:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
C220-WZP28359K2C /bios # show detail
BIOS:
    <mark>BIOS Version: C220M7.4.3.5b.0_XRV9K</mark>
    Backup BIOS Version: C220M7.4.3.5a.0_XRV9K
    Boot Order: CDROM,HDD
    FW Update Status: Done, OK
    UEFI Secure Boot: enabled
    Actual Boot Mode: Uefi
    Last Configured Boot Order Source: CIMC
    One time boot device: (none)
</code>
</pre>
</div>    

# Conclusion

There are fundamentally very few differences between upgrading a traditional ASR 9000 router and the IOS XRv 9000 UCS M7 appliance: both run the same IOS XR 64bit architecture.

- For IOS XR software upgrades, users have the choice between the traditional install add/activate/commit picking up individual mini ISO and optional RPMs (packages, SMUs) or simply use install replace with a golden ISO (custom built or Cisco provided)
- For CIMC, BIOS & firmwares upgrade, users must currently rely on CIMC (via GUI or CLI) while waiting for IOS XR improvement to package those.

Users are invited to combine both IOS XR and firmware upgrades in the same maintenance window, and run a Cisco supported combination.
