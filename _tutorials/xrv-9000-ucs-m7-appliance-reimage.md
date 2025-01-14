---
published: true
date: '2025-01-13 14:26 +0100'
title: Cisco IOS XRv 9000 UCS M7 Appliance reimage procedure
author: Frederic Cuiller
tags:
  - iosxr
  - Appliance
  - XRv 9000
position: hidden
excerpt: >-
  This procedure aims to document the steps required to fully reimage a Cisco
  IOS XRv 9000 Appliance running IOS XR 24.4.1 and based on Cisco UCS M7 series.
---
{% include toc icon="table" title="Cisco IOS XRv 9000 UCS M7 reimage procedure" %}

# Objectives
This procedure aims to document the steps required to fully reimage a Cisco IOS XRv 9000 Appliance based on UCS M7 C220 series running IOS XR 24.4.1. 
Reimage can be used to fully reinstall the system from scratch for staging or system recovery purposes. 
Reimage will remove all files and configuration from the system. It's important to perform backup before executing this procedure.

# Cisco IOS XRv 9000 UCS M7 appliance introduction

The Cisco IOS XRv 9000 UCS M7 appliance is powered by Cisco UCS C220 M7 hardware and Cisco IOS-XR 64bit software. It is suitable for BGP Route-Reflector (RR) and Path Computation Element (PCE) use cases.

There are two references available:
- XRV-M7-APLN-25G: 4x10G/25G ports available on a single NIC (Cisco-Intel E810XXVDA4L 4x25/10 GbE SFP28 PCIe NIC)
- XRV-M7-APLN-100G: 4x100G ports available on 2 x NICs (Cisco-MLNX MCX623106AS-CDAT 2x100GbE QSFP56 PCIe NIC)

Both servers run the same CPU, Memory, Disk & Management complex:
- Intel I5420+ CPU, 28 cores running at 2GHz
- 128GB DDR5 RAM
- 480GB SSD
- Intel X710-T2L NIC for OCP 3.0

# Prerequisites
It’s important to check UCS appliance is healthy before proceeding further. This can be verified on the CIMC interface and no faults should be reported:
![UCS-CIMC-precheck.png]({{site.baseurl}}/images/UCS-CIMC-precheck.png)

Review faults and logs and ensure there is anything suspicious:
![UCS-CIMC-precheck.png]({{site.baseurl}}/images/UCS-CIMC-precheck.png)

In case of doubt, please open a TAC case.

# Cisco IOS XRv 9000 ISO Download
IOS XR 24.4.1 image can be downloaded from [this](https://software.cisco.com/download/home/286288939/type/280805694) location. The file fullk9-R-XRV9000-2441-RR.tar must be used: it contains k9 package for crypto features (e.g SSH), and the image is optimized for a BGP Route-Reflector function: 72GB of RAM is dedicated to the virtual Route Processor, 16GB for the virtual Line Card.  

Once extracted, following files are available:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
{25-01-09 22:15}FCUILLER-M-X6F3:~/Downloads/ISO/fullk9-R-XRV9000-2441-RR fcuiller% tree .
.
├── README-fullk9-R-XRV9000-2441-RR.txt
├── xrv9k-bng-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-bng-supp-x64-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-eigrp-1.0.0.0-r2441.x86_64.rpm
<mark>├── xrv9k-fullk9-x.vrr-24.4.1.iso</mark>
├── xrv9k-isis-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-k9sec-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-li-x-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-m2m-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-mcast-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-mgbl-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-mini-x-24.4.1.iso
├── xrv9k-mpls-1.0.0.0-r2441.x86_64.rpm
├── xrv9k-mpls-te-rsvp-1.0.0.0-r2441.x86_64.rpm
└── xrv9k-ospf-1.0.0.0-r2441.x86_64.rpm

0 directories, 15 files
{25-01-09 22:15}FCUILLER-M-X6F3:~/Downloads/ISO/fullk9-R-XRV9000-2441-RR fcuiller% 
</code>
</pre>
</div>

To execute the reimage procedure, xrv9k-fullk9-x.vrr-24.4.1.iso will be used.

# Virtual Media Mapping

The ISO file is mapped to a virtual CD/DVD which will be later used as a boot device. To do so, click on the _Remote Management Properties_ button in the KVM Console window:

![UCS-CIMC-properties.png]({{site.baseurl}}/images/UCS-CIMC-properties.png)

This opens a new window with a list of current mappings. This list should be empty by default:

![UCS-CIMC-current-mapping.png]({{site.baseurl}}/images/UCS-CIMC-current-mapping.png)

Click on the _Edit Configuration_ button, this will open a new Remote Management window:

![UCS-CIMC-add-mapping.png]({{site.baseurl}}/images/UCS-CIMC-add-mapping.png)

From here, a new mapping can be added. Click on _Add New Mapping_ button. Populate the fields to upload the ISO to the CIMC using your favorite protocol. The Virtual Media Type must be CD:

![UCS-CIMC-virtualCD.png]({{site.baseurl}}/images/UCS-CIMC-virtualCD.png)

A message indicates the successful creation of the virtual media, which can be seen as mapped:

![UCS-CIMC-virtualCD-success.png]({{site.baseurl}}/images/UCS-CIMC-virtualCD-success.png)

**Note:** If CIMC cannot be used to create the virtual media, the vKVM allows to create a vKVM-Mapped vDVD which can uploaded from the user web browser directly.
![UCS-CIMC-vDVD-vKVM.png]({{site.baseurl}}/images/UCS-CIMC-vDVD-vKVM.png)
{: .notice--primary}

# Boot Order Update

Next step is to change the boot order sequence. By default, appliance will boot on harddisk (GRUB hd0) as seen from the KVM:
![UCS-CIMC-KVM-hd0.png]({{site.baseurl}}/images/UCS-CIMC-KVM-hd0.png)

Boot Order can be updated using the KVM Console and by clicking on _Launch KVM_ button:
![UCS-CIMC-launchKVM.png]({{site.baseurl}}/images/UCS-CIMC-launchKVM.png)

Once on the KVM, select Boot Device menu, and click on _CIMC Mapped vDVD_ button:
![UCS-CIMC-vDVDmapped.png]({{site.baseurl}}/images/UCS-CIMC-vDVDmapped.png)

# Appliance Reboot

At this stage, appliance must be rebooted. This is achieved with the Power menu, and by clicking on _Reset System_ button:
![UCS-CIMC-reset.png]({{site.baseurl}}/images/UCS-CIMC-reset.png)

UCS will reload, perform hardware sanity check will use CD1 as a boot media:
![UCS-CIMC-bootCD.png]({{site.baseurl}}/images/UCS-CIMC-bootCD.png)

On the console, following logs can be observed. Existing system is totally wiped: all partitions and their files are removed and a new filesystem is created:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
<mark>Booting from cdrom..</mark>
Loading Kernel..
Verifying (loop)/boot/bzImage...
(loop)/boot/bzImage verified using attached signature.
Verifying (loop)/boot/bzImage...
(loop)/boot/bzImage verified using attached signature.
Loading initrd..
Verifying (loop)/boot/initrd.img...
(loop)/boot/initrd.img verified using Pkcs7 signature.
[   24.579977] integrity: Problem loading X.509 certificate -126
Adding sub-CA and leaf certificates to keyring

Welcome to Linux Distro for XR 11.1.2 (dunfell)!

[  OK  ] Created slice Virtual Machine and Container Slice.
[  OK  ] Created slice system-getty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[  OK  ] Created slice User and Session Slice.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[  OK  ] Reached target Paths.
[  OK  ] Reached target Remote File Systems.
[  OK  ] Reached target Slices.
[  OK  ] Reached target Swap.
[  OK  ] Reached target Libvirt guests shutdown.
[  OK  ] Listening on Device-mapper event daemon FIFOs.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Reached target RPC Port Mapper.
[  OK  ] Listening on Syslog Socket.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket.
[  OK  ] Listening on Network Service Netlink Socket.
[  OK  ] Listening on udev Control Socket.
[  OK  ] Listening on udev Kernel Socket.
         Mounting Huge Pages File System...
         Mounting POSIX Message Queue File System...
         Mounting Kernel Debug File System...
         Mounting Temporary Directory (/tmp)...
         Starting Availability of block devices...
         Starting Create list of st…odes for the current kernel...
         Starting Monitoring of LVM…meventd or progress polling...
         Starting RPC Bind...
         Starting SELinux autorelabel service loading...
         Starting SELinux init for /dev service loading...
         Starting Journal Service...
         Starting Load Kernel Modules...
         Starting Remount Root and Kernel File Systems...
         Starting udev Coldplug all Devices...
[  OK  ] Started RPC Bind.
[  OK  ] Started Journal Service.
[  OK  ] Mounted Huge Pages File System.
[  OK  ] Mounted POSIX Message Queue File System.
[  OK  ] Mounted Kernel Debug File System.
[  OK  ] Mounted Temporary Directory (/tmp).
[  OK  ] Started Availability of block devices.
[  OK  ] Started Create list of sta… nodes for the current kernel.
[  OK  ] Started SELinux init for /dev service loading.
[  OK  ] Started Load Kernel Modules.
[  OK  ] Started Remount Root and Kernel File Systems.
         Mounting FUSE Control File System...
         Starting Flush Journal to Persistent Storage...
         Starting Apply Kernel Variables...
         Starting Create System Users...
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Started Flush Journal to Persistent Storage.
[  OK  ] Started Apply Kernel Variables.
[  OK  ] Started Create System Users.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Started Create Static Device Nodes in /dev.
         Starting udev Kernel Device Manager...
[  OK  ] Started udev Kernel Device Manager.
[  OK  ] Found device /dev/ttyS0.
[  OK  ] Started udev Coldplug all Devices.
[  OK  ] Created slice system-lvm2\x2dpvscan.slice.
         Starting LVM event activation on device 8:3...
         Starting LVM event activation on device 8:5...
         Starting udev Wait for Complete Device Initialization...
[  OK  ] Started LVM event activation on device 8:5.
[  OK  ] Started Monitoring of LVM2… dmeventd or progress polling.
[  OK  ] Started LVM event activation on device 8:3.
[  OK  ] Reached target Local File Systems (Pre).
         Mounting /mnt...
         Mounting /var/volatile...
[  OK  ] Mounted /mnt.
[  OK  ] Mounted /var/volatile.
         Starting Load/Save Random Seed...
[  OK  ] Reached target Local File Systems.
         Starting Rebuild Dynamic Linker Cache...
         Starting SELinux init service loading...
         Starting Create Volatile Files and Directories...
[  OK  ] Started Load/Save Random Seed.
[  OK  ] Started Rebuild Dynamic Linker Cache.
[  OK  ] Started SELinux init service loading.
[  OK  ] Started Create Volatile Files and Directories.
         Starting Security Auditing Service...
         Starting Run pending postinsts...
         Starting Rebuild Journal Catalog...
[  OK  ] Started Security Auditing Service.
[  OK  ] Started Rebuild Journal Catalog.
         Starting Update is Completed...
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started Update is Completed.
[  OK  ] Started Run pending postinsts.
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Started udev Wait for Complete Device Initialization.
[  OK  ] Started Hardware RNG Entropy Gatherer Daemon.
[  OK  ] Started SELinux autorelabel service loading.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Periodic rotation of log files.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target Timers.
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Listening on Libvirt local socket.
[  OK  ] Listening on Libvirt admin socket.
[  OK  ] Listening on Libvirt local read-only socket.
[  OK  ] Listening on Tftp Server Activation Socket.
[  OK  ] Listening on Virtual machine lock manager socket.
[  OK  ] Listening on Virtual machine log manager socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
         Starting Avahi mDNS/DNS-SD Stack...
[  OK  ] Started Periodic Command Scheduler.
[  OK  ] Started D-Bus System Message Bus.
         Starting Ethernet Bridge Filtering Tables...
         Starting IPv6 Packet Filtering Framework...
         Starting IPv4 Packet Filtering Framework...
         Starting Lighttpd Daemon...
[  OK  ] Started Machine Check Exception Logging Daemon.
[  OK  ] Started Cisco Process Monitor Service.
         Starting Rollback uncommit… config change transactions...
         Starting System Logging Service...
[  OK  ] Started Self Monitoring an…ing Technology (SMART) Daemon.
         Starting Login Service...
         Starting Virtual Machine a…tainer Registration Service...
         Starting XR installation service...
         Starting OpenSSH Key Generation...
[  OK  ] Started System Logging Service.
[  OK  ] Started Ethernet Bridge Filtering Tables.
[  OK  ] Started IPv6 Packet Filtering Framework.
[  OK  ] Started IPv4 Packet Filtering Framework.
[  OK  ] Started Lighttpd Daemon.
[  OK  ] Started Rollback uncommitt…rk config change transactions.
[  OK  ] Reached target Network (Pre).
         Starting Network Service...
         Starting Rotate log files...
[  OK  ] Started Network Service.
[  OK  ] Started Rotate log files.
[  OK  ] Started Virtual Machine an…ontainer Registration Service.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
[  OK  ] Started Login Service.
[  OK  ] Reached target Network.
         Starting Create Network Namespaces...
         Starting Virtualization daemon...
[  OK  ] Started Respond to IPv6 Node Information Queries.
[  OK  ] Started Network Router Discovery Daemon.
[  OK  ] Started Redis In-Memory Data Store.
         Starting Permit User Sessions...
[  OK  ] Started Xinetd A Powerful Replacement For Inetd.
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Reached target Login Prompts.
[  OK  ] Stopped Redis In-Memory Data Store.
[  OK  ] Started Redis In-Memory Data Store.
         Starting Rotate log files...
[  OK  ] Started Rotate log files.
         Starting XR PXE Installation service...
[   32.826344] pxe_install.sh[1704]: Logging started at Thu Jan  9 13:44:09 UTC 2025...
[  OK  ] Started OpenSSH Key Generation.
         Starting OpenSSH server daemon...
[  OK  ] Started OpenSSH server daemon.
[  OK  ] Stopped Redis In-Memory Data Store.
[  OK  ] Started Redis In-Memory Data Store.
[  OK  ] Stopped Redis In-Memory Data Store.
[  OK  ] Started Redis In-Memory Data Store.
[   33.451255] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Sourcing /etc/init.d/pi_grub_and_menu_lst_update.sh PI script
[   33.466037] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Sourcing /etc/init.d/mount_pd_fs.sh PI script
[   33.480055] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: /dev/sda disk size is 240057MB
[   33.492085] pxe_install.sh[1720]: cmdline = BOOT_IMAGE=(loop)/boot/bzImage __reboot_on_xr_bake=true __hugepages=3072 __hw_profile=vrr root=/dev/ram install=/dev/sda console=ttyS0,115200 prod=1 crashkernel=192M@0 bigphysarea=10M quiet pci=assign-busses noissu aer=off pci=hpmemsize=0M,hpiosize=0M net.ifnames=0
[   33.524043] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Preparing disk for PLATFORM=BOOT_IMAGE=(loop)/boot/bzImage:
[   33.538030] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: In collect_archive_interrupted_bake_logs function....
[   33.552032] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Secondary disk is not present
[   33.565033] [pxe_install.sh  OK  [1704]: ] Thu Jan  9 13:44:10 UTC 2025: Installer will install image on sdaStarted Virtualization daemon.

[   33.565079] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Checking disks: sda
         Starting Suspend/Resume Running libvirt Guests...
[  OK  ] Stopped Redis In-Memory Data Store.
[  OK  ] Started Redis In-Memory Data Store.
[  OK  ] Started Suspend/Resume Running libvirt Guests.
<mark>[   33.623244] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Removing old boot partition</mark>
<mark>[   33.642069] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Removing old volumes</mark>
[   33.652032] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Inside Volume-Cleaning Function
[  OK  ] Stopped Redis In-Memory Data Store.
[  OK  ] Started Redis In-Memory Data Store.
[   33.779803] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Removed LVM /dev/panini_vol_grp/host_lv0 for Panini
[   33.862587] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Removed LVM /dev/panini_vol_grp/host_data_scratch_lv0 for Panini
[  OK  ] Stopped Redis In-Memory Data Store.
[FAILED] Failed to start Redis In-Memory Data Store.
See 'systemctl status redis.service' for details.
[   33.943687] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Removed LVM /dev/panini_vol_grp/host_data_log_lv0 for Panini
[   34.037591] pxe_install.sh[1704]: Thu Jan  9 13:44:10 UTC 2025: Removed LVM /dev/panini_vol_grp/host_data_config_lv0 for Panini
[   34.123608] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/calvados_lv0 for Panini
[   34.211400] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/calvados_data_lv0 for Panini
[   34.293536] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/ssd_disk1_calvados_1 for Panini
[   34.372569] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/ssd_disk1_xr_1 for Panini
[   34.456500] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/ssd_disk1_calvados_swtam_1 for Panini
[   34.539710] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/xr_lv0 for Panini
[   34.631103] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/xr_data_lv0 for Panini
[   34.744560] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/xr_lcp_lv0 for Panini
[  OK  ] Started Create Network Namespaces.
[   34.848519] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/panini_vol_grp/xr_lcp_data_lv0 for Panini
[   34.960464] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed LVM /dev/app_vol_grp/app_lv0 for App-Host
[   35.061498] pxe_install.sh[1704]: Thu Jan  9 13:44:11 UTC 2025: Removed App-Vol Grp app_vol_grp
[   35.178491] pxe_install.sh[1704]: Thu Jan  9 13:44:12 UTC 2025: Removed Panini Vol-Grp panini_vol_grp
[  OK  ] Started /usr/sbin/lvm pvscan --cache 8:3.
[  OK  ] Started /usr/sbin/lvm pvscan --cache 8:5.
[   35.429687] pxe_install.sh[1704]: Thu Jan  9 13:44:12 UTC 2025: Removed PV
[   35.439043] pxe_install.sh[1704]: Thu Jan  9 13:44:12 UTC 2025: Exiting from the Volume Cleaning Section
[   36.604856] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: Md5sum verification succeeded
[   36.617050] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: ---Starting to prepare mb_disk---
[   36.630027] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: ---Preparing partition for bios logging---
[   36.642026] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025:
[   36.651028] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: ---Starting to prepare sda---
         Stopping LVM event activation on device 8:3...
         Stopping LVM event activation on device 8:5...
[   36.990685] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: LVM size (178089 MB) DEBUG part size (0 MB)
[  OK  ] Stopped LVM event activation on device 8:3.
<mark>[   37.010105] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: Creating partitions, BOOT=944MB, LVM=178089MB, EFI=20MB</mark>
[  OK  ] Stopped LVM event activation on device 8:5.
[   37.059597] pxe_install.sh[1704]: Thu Jan  9 13:44:13 UTC 2025: Created Partition of size 4096 for use of App-Volume
[   37.109452] pxe_install.sh[1704]: Thu Jan  9 13:44:14 UTC 2025: Partition creation on /dev/sda took 1 seconds
[   37.244590] pxe_install.sh[1704]: Thu Jan  9 13:44:14 UTC 2025: File system creation on /dev/sda1 took 0 seconds
[   37.258060] pxe_install.sh[1704]: Thu Jan  9 13:44:14 UTC 2025: Install boot image on /dev/sda1

Linux Distro for XR 11.1.2 ios ttyS0

ios login:
Linux Distro for XR 11.1.2 ios ttyS0

ios login: [   43.856589] pxe_install.sh[1720]: Thu Jan  9 13:44:20 UTC 2025: Starting Calvados patch for lxc for hostos
[   43.857076] pxe_install.sh[1720]: Thu Jan  9 13:44:20 UTC 2025: Uninstalling rpm gdb
[   43.940203] pxe_install.sh[1720]: SELinux: /.autorelabel placed, filesystem will be relabeled...
[   43.977012] pxe_install.sh[1720]: Cleaning out /tmp
[   43.979153] pxe_install.sh[1720]: Relabeling / /dev /sys
/ 100.0%96318] pxe_install.sh[1720]:
/dev 100.0%34] pxe_install.sh[1720]:
/sys 100.0%69] pxe_install.sh[1720]:
[   49.327060] pxe_install.sh[1720]: Cleaning up labels on /tmp
[   49.334992] pxe_install.sh[1720]:  * Relabel done.
[   49.339100] pxe_install.sh[1720]: Thu Jan  9 13:44:26 UTC 2025: Finished Calvados patch for lxc
<mark>[   49.361821] pxe_install.sh[1720]: Thu Jan  9 13:44:26 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): force hw_profile to be vrr</mark>
[   49.372734] pxe_install.sh[1720]: Thu Jan  9 13:44:26 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Disabling SMT
[   50.007607] pxe_install.sh[1720]: Thu Jan  9 13:44:26 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Based on system memory and socket number, use the following huge page setting:
[   50.010732] pxe_install.sh[1720]: Thu Jan  9 13:44:26 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh):     default_hugepagesz=1G hugepagesz=1G hugepages=6
[   50.489451] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Isolate CPUs (for Data Plane) Setting: isolcpus=22-27
[   50.517146] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: -- default 0
[   50.517208] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: -- timeout 5
serial --unit=0 --speed=1152001720]: Thu Jan  9 13:44:27 UTC 2025: --
[   50.517273] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: -- terminal --timeout=5 console serial
[   50.517288] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: --
[   50.517301] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: -- title xrv9000
[   50.517315] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: --    root (hd0,0)
[   50.517335] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: --   kernel /boot/bzImage root=/dev/panini_vol_grp/host_lv0 __reboot_on_xr_bake=true __hw_profile=vrr intel_iommu=off pcie_aspm=off platform=xrv9k isolcpus=22-27 default_hugepagesz=1G hugepagesz=1G hugepages=6 nosmt=force elevator=noop boardtype=RP vmtype=hostos console=ttyS0 prod=1 crashkernel=192M@0 bigphysarea=10M quiet pci=hpmemsize=0M,hpiosize=0M
[   50.517351] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025: --   initrd /boot/initrd.img
[   50.844211] pxe_install.sh[1704]: Thu Jan  9 13:44:27 UTC 2025: Installing host image size of 780M took 13 seconds
[   50.844835] pxe_install.sh[1704]: Thu Jan  9 13:44:27 UTC 2025:
[   50.845501] pxe_install.sh[1704]: Thu Jan  9 13:44:27 UTC 2025: ---Starting to prepare host logical volume---
[   50.890113] pxe_install.sh[1720]: Thu Jan  9 13:44:27 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Create app volume on sda
[   50.980507] pxe_install.sh[1720]:   Physical volume "/dev/sda5" successfully created.
[   51.494822] pxe_install.sh[1720]: Thu Jan  9 13:44:28 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Create app volume repo.
[   59.090112] pxe_install.sh[1720]: Thu Jan  9 13:44:36 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): XRv9k: card_inst=XRV9K-D-RP
[   59.519556] pxe_install.sh[1720]: Thu Jan  9 13:44:36 UTC 2025: Starting Calvados patch for lxc for hostos
[   59.894507] pxe_install.sh[1720]: Thu Jan  9 13:44:36 UTC 2025: Uninstalling rpm gdb
[   59.913739] pxe_install.sh[1720]: Thu Jan  9 13:44:36 UTC 2025 (/etc/init.d/calvados_patch_lxc_iso.sh): xrv9000: Patch host
[   59.932651] pxe_install.sh[1720]: Thu Jan  9 13:44:36 UTC 2025 (/etc/init.d/calvados_patch_lxc_iso.sh): Disable DHCP on host eth, host-eth0
[   59.977750] pxe_install.sh[1720]: mount: /tmp/bootdisk: no medium found on /dev/sr0.
[   59.986583] pxe_install.sh[1720]: /dev/null already exists.
[   59.997780] pxe_install.sh[1720]: Thu Jan  9 13:44:36 UTC 2025 (/etc/init.d/calvados_patch_lxc_iso.sh): xrv9000: Patch host done
[   60.089584] pxe_install.sh[1720]: SELinux: /.autorelabel placed, filesystem will be relabeled...
[   60.130476] pxe_install.sh[1720]: Cleaning out /tmp
[   60.132564] pxe_install.sh[1720]: Relabeling / /dev /sys
/ 100.0%84358] pxe_install.sh[1720]:
/dev 100.0%07] pxe_install.sh[1720]:
/sys 100.0%30] pxe_install.sh[1720]:
[   65.916253] pxe_install.sh[1720]: Cleaning up labels on /tmp
[   65.924644] pxe_install.sh[1720]:  * Relabel done.
[   65.928977] pxe_install.sh[1720]: Thu Jan  9 13:44:42 UTC 2025: Finished Calvados patch for lxc
[   66.468338] pxe_install.sh[1704]: Thu Jan  9 13:44:43 UTC 2025:
[   66.470210] pxe_install.sh[1704]: Thu Jan  9 13:44:43 UTC 2025: ---Starting to prepare calvados logical volume---
[   66.582646] pxe_install.sh[1704]: Thu Jan  9 13:44:43 UTC 2025: Create sub partition on /dev/panini_vol_grp/calvados_lv0
[   66.801062] pxe_install.sh[1704]: Thu Jan  9 13:44:43 UTC 2025: Create data sub partition on /dev/panini_vol_grp/calvados_data_lv0
[   68.672590] pxe_install.sh[1704]: Thu Jan  9 13:44:45 UTC 2025: File system creation on /dev/panini_vol_grp/calvados_lv0 took 2 seconds
[   68.721149] pxe_install.sh[1704]: Thu Jan  9 13:44:45 UTC 2025: Install sysadmin-vm image on /dev/panini_vol_grp/calvados_lv0
[   71.390724] pxe_install.sh[4220]: Thu Jan  9 13:44:48 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): XRv9k: card_inst=XRV9K-D-RP
[   71.398816] pxe_install.sh[1704]: Thu Jan  9 13:44:48 UTC 2025: sysadmin-vm: RP based installation
[   73.769654] pxe_install.sh[4220]: /dev/null not found. Creating it.
[   74.896741] pxe_install.sh[4220]: '/usr/lib64/python3.7/site-packages//tmp_staging/./calv_instcmd_pb2.py' -> '/usr/lib64/python3.7/site-packages/./calv_instcmd_pb2.py'
snip
[   79.764314] pxe_install.sh[4220]: Thu Jan  9 13:44:56 UTC 2025: Starting Calvados patch for lxc for sysadmin-vm
[   79.949790] pxe_install.sh[4220]: Thu Jan  9 13:44:56 UTC 2025: Uninstalling rpm gdb
[   79.953950] pxe_install.sh[4220]: Thu Jan  9 13:44:56 UTC 2025: Uninstalling rpm smartmontools
[   80.015388] pxe_install.sh[4220]: Thu Jan  9 13:44:56 UTC 2025 (/etc/init.d/calvados_patch_lxc_iso.sh): xrv9000: Patch calvados
[   80.037863] pxe_install.sh[4220]: /dev/null already exists.
[   80.045432] pxe_install.sh[4220]: Thu Jan  9 13:44:56 UTC 2025 (/etc/init.d/calvados_patch_lxc_iso.sh): xrv9000: Patch calvados done
[   80.130546] pxe_install.sh[4220]: SELinux: /.autorelabel placed, filesystem will be relabeled...
[   80.171130] pxe_install.sh[4220]: Cleaning out /tmp
[   80.173459] pxe_install.sh[4220]: Relabeling / /dev /sys
/ 100.0%41601] pxe_install.sh[4220]:
/dev 100.0%78] pxe_install.sh[4220]:
/sys 100.0%67] pxe_install.sh[4220]:
[   85.974566] pxe_install.sh[4220]: Cleaning up labels on /tmp
[   85.983260] pxe_install.sh[4220]:  * Relabel done.
[   85.986851] pxe_install.sh[4220]: Thu Jan  9 13:45:02 UTC 2025: Finished Calvados patch for lxc
[   86.347639] pxe_install.sh[1704]: Thu Jan  9 13:45:03 UTC 2025: Installing sysadmin-vm image size of 678M took 18 seconds
[   88.484991] pxe_install.sh[1704]: Thu Jan  9 13:45:05 UTC 2025: ---Starting to prepare repository---
[   89.951533] pxe_install.sh[1704]: Thu Jan  9 13:45:06 UTC 2025: File system creation on /dev/sda2 took 1 seconds
[   92.007025] pxe_install.sh[4220]: Thu Jan  9 13:45:08 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Check for unwanted iso and remove if required.
[   92.046061] pxe_install.sh[1704]: Thu Jan  9 13:45:08 UTC 2025: Copying /iso/host.iso to repository /iso directory
[   92.153564] pxe_install.sh[1704]: Thu Jan  9 13:45:09 UTC 2025: Copying /iso/xrv9k-sysadmin.iso to repository /iso directory
[   93.378693] pxe_install.sh[1704]: Thu Jan  9 13:45:10 UTC 2025: Copying /iso/xrv9k-xr.iso to repository /iso directory
[   97.095948] pxe_install.sh[1704]: Thu Jan  9 13:45:14 UTC 2025: Copying all ISOs to repository took 6 seconds
[   99.263893] pxe_install.sh[1704]: Thu Jan  9 13:45:16 UTC 2025: Install EFI on /dev/sda4
[   99.292131] pxe_install.sh[4220]: Not Found: /tmp/signed_grub.MB9FtF/EFI/boot/grubenv
[   99.293764] pxe_install.sh[4220]: Grub not signed. Continuing with traditional grub ...
[   99.367435] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): force hw_profile to be vrr
[   99.389060] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Disabling SMT
[   99.423640] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Based on system memory and socket number, use the following huge page setting:
[   99.427297] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh):     default_hugepagesz=1G hugepagesz=1G hugepages=6
[   99.920580] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Isolate CPUs (for Data Plane) Setting: isolcpus=22-27
[   99.957751] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- set default=0
[   99.957799] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --
[   99.957816] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- serial --unit=0 --speed=115200
[   99.957829] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --
[   99.957840] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- terminal_input console
[   99.957852] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- terminal_output serial
[   99.957863] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --
[   99.957874] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- set timeout=2
[   99.957886] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --
[   99.957935] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- menuentry "System Host OS" {
[   99.957947] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --     echo "Booting from Disk.."
[   99.957959] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --     search.fs_label HostOs root
[   99.957971] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --     echo "Loading Kernel.."
[   99.957987] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --     linux  /boot/bzImage root=/dev/panini_vol_grp/host_lv0 __reboot_on_xr_bake=true __hw_profile=vrr intel_iommu=off pcie_aspm=off platform=BOOT_IMAGE=(loop)/boot/bzImage __reboot_on_xr_bake=true __hw_profile=vrr platform=xrv9k isolcpus=22-27 default_hugepagesz=1G hugepagesz=1G hugepages=6 nosmt=force elevator=noop boardtype=RP vmtype=hostos ima_tcb ima_appraise=log evm=off console=ttyS0,115200 prod=1 crashkernel=400M@0 bigphysarea=10M quiet pci=assign-busses  aer=off pci=hpmemsize=0M,hpiosize=0M enable_efi_vars=0 pcie_port_pm=off net.ifnames=0
[   99.958437] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --     echo "Loading initrd.."
[   99.958452] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --     initrd /boot/initrd.img
[   99.958465] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: -- }
[   99.958477] pxe_install.sh[4220]: Thu Jan  9 13:45:16 UTC 2025: --
[   99.974501] pxe_install.sh[1704]: Thu Jan  9 13:45:16 UTC 2025: Adding sda4 to efi boot option
[   99.996711] pxe_install.sh[4220]: BootCurrent: 0012
[   99.996732] pxe_install.sh[4220]: Timeout: 3 seconds
[   99.996748] pxe_install.sh[4220]: BootOrder: 0012,0001,000D,0011
[   99.996798] pxe_install.sh[4220]: Boot0001* UEFI: Built-in EFI Shell
[   99.996814] pxe_install.sh[4220]: Boot000D* UEFI: PXE IPv4 Cisco(R) Ethernet Network Adapter X710-T2L OCP 3.0
[   99.996829] pxe_install.sh[4220]: Boot0011* UEFI: PXE IPv4 Cisco(R) X710TLG GbE RJ45 PCIe NIC
[   99.996844] pxe_install.sh[4220]: Boot0012* UEFI: Cisco CIMC-Mapped vDVD2.00
[   99.997072] pxe_install.sh[4220]: MirroredPercentageAbove4G: 0.00
[   99.997087] pxe_install.sh[4220]: MirrorMemoryBelow4GB: false
[  100.015590] pxe_install.sh[4220]: BootCurrent: 0012
[  100.015607] pxe_install.sh[4220]: Timeout: 3 seconds
[  100.015621] pxe_install.sh[4220]: BootOrder: 0000,0012,0001,000D,0011
[  100.015636] pxe_install.sh[4220]: Boot0001* UEFI: Built-in EFI Shell
[  100.015690] pxe_install.sh[4220]: Boot000D* UEFI: PXE IPv4 Cisco(R) Ethernet Network Adapter X710-T2L OCP 3.0
[  100.015706] pxe_install.sh[4220]: Boot0011* UEFI: PXE IPv4 Cisco(R) X710TLG GbE RJ45 PCIe NIC
[  100.015721] pxe_install.sh[4220]: Boot0012* UEFI: Cisco CIMC-Mapped vDVD2.00
[  100.015735] pxe_install.sh[4220]: Boot0000* CiscoXR
[  100.015929] pxe_install.sh[4220]: MirroredPercentageAbove4G: 0.00
[  100.015943] pxe_install.sh[4220]: MirrorMemoryBelow4GB: false
[  100.034657] pxe_install.sh[4220]: BootNext: 0000
[  100.034673] pxe_install.sh[4220]: BootCurrent: 0012
[  100.034686] pxe_install.sh[4220]: Timeout: 3 seconds
[  100.034698] pxe_install.sh[4220]: BootOrder: 0000,0012,0001,000D,0011
[  100.034712] pxe_install.sh[4220]: Boot0000* CiscoXR
[  100.034725] pxe_install.sh[4220]: Boot0001* UEFI: Built-in EFI Shell
[  100.034739] pxe_install.sh[4220]: Boot000D* UEFI: PXE IPv4 Cisco(R) Ethernet Network Adapter X710-T2L OCP 3.0
[  100.034752] pxe_install.sh[4220]: Boot0011* UEFI: PXE IPv4 Cisco(R) X710TLG GbE RJ45 PCIe NIC
[  100.034765] pxe_install.sh[4220]: Boot0012* UEFI: Cisco CIMC-Mapped vDVD2.00
[  100.034871] pxe_install.sh[4220]: MirroredPercentageAbove4G: 0.00
[  100.034885] pxe_install.sh[4220]: MirrorMemoryBelow4GB: false
[  100.038027] pxe_install.sh[1704]: Thu Jan  9 13:45:16 UTC 2025: Install finished on sda
<mark>[  100.051689] pxe_install.sh[11564]: Thu Jan  9 13:45:16 UTC 2025 (/etc/rc.d/init.d/pxe_install.sh): Eject CDROM and reboot XRv9k system after installation ...</mark>
�����������������������������������������������������
</code>
</pre>
</div>

**Note:** At this point, the virtual media must be ejected. Click on the Virtual Media Menu, and select the ISO:
![UCS-CIMC-ejectCD.png]({{site.baseurl}}/images/UCS-CIMC-ejectCD.png)
And confirm:
![UCS-CIMC-ejectCD-confirm.png]({{site.baseurl}}/images/UCS-CIMC-ejectCD-confirm.png)
{: .notice--primary}

The appliance then reboots from the HDD.
Following logs can be observed on the console:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
<mark>  Booting `System Host OS'

Booting from Disk..</mark>
Loading Kernel..
Verifying /boot/bzImage...
/boot/bzImage verified using attached signature.
Verifying /boot/bzImage...
/boot/bzImage verified using attached signature.
Loading initrd..
Verifying /boot/initrd.img...
/boot/initrd.img verified using Pkcs7 signature.
[   16.871497] integrity: Unable to open file: /etc/keys/x509_ima.der (-2)
Adding sub-CA and leaf certificates to keyring
XRv9K init
Cleaning up with exit code 0 ...
Switching to new root and running init.

Welcome to Linux Distro for XR 11.1.2 (dunfell)!

[  OK  ] Created slice Virtual Machine and Container Slice.
[  OK  ] Created slice User and Session Slice.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[  OK  ] Reached target Paths.
[  OK  ] Reached target Remote File Systems.
[  OK  ] Reached target Slices.
[  OK  ] Reached target Swap.
[  OK  ] Reached target Libvirt guests shutdown.
[  OK  ] Listening on Device-mapper event daemon FIFOs.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Reached target RPC Port Mapper.
[  OK  ] Listening on Syslog Socket.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket.
[  OK  ] Listening on Network Service Netlink Socket.
[  OK  ] Listening on udev Control Socket.
[  OK  ] Listening on udev Kernel Socket.
         Mounting Huge Pages File System...
         Mounting POSIX Message Queue File System...
         Mounting Kernel Debug File System...
         Mounting Temporary Directory (/tmp)...
         Starting Availability of block devices...
         Starting Create list of st…odes for the current kernel...
         Starting Monitoring of LVM…meventd or progress polling...
         Starting RPC Bind...
         Starting SELinux autorelabel service loading...
         Starting SELinux init for /dev service loading...
         Starting Spirit early boot setup...
         Starting Journal Service...
         Starting Load Kernel Modules...
         Starting Remount Root and Kernel File Systems...
         Starting udev Coldplug all Devices...
[  OK  ] Started RPC Bind.
[  OK  ] Mounted Huge Pages File System.
[  OK  ] Mounted POSIX Message Queue File System.
[  OK  ] Mounted Kernel Debug File System.
[  OK  ] Mounted Temporary Directory (/tmp).
[  OK  ] Started Availability of block devices.
[  OK  ] Started Create list of sta… nodes for the current kernel.
[  OK  ] Started Journal Service.
[  OK  ] Started SELinux autorelabel service loading.
[  OK  ] Started SELinux init for /dev service loading.
[  OK  ] Started Remount Root and Kernel File Systems.
         Starting Flush Journal to Persistent Storage...
         Starting Create System Users...
[  OK  ] Started Flush Journal to Persistent Storage.
[  OK  ] Started Create System Users.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Started udev Coldplug all Devices.
         Starting udev Wait for Complete Device Initialization...
[  OK  ] Started Load Kernel Modules.
         Mounting FUSE Control File System...
         Starting Apply Kernel Variables...
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Started Create Static Device Nodes in /dev.
         Starting udev Kernel Device Manager...
[  OK  ] Started Apply Kernel Variables.
[  OK  ] Started udev Kernel Device Manager.
[  OK  ] Created slice system-lvm2\x2dpvscan.slice.
         Starting LVM event activation on device 8:3...
         Starting LVM event activation on device 8:5...
[  OK  ] Started LVM event activation on device 8:3.
[  OK  ] Started LVM event activation on device 8:5.
[  OK  ] Started Monitoring of LVM2… dmeventd or progress polling.
[  OK  ] Reached target Local File Systems (Pre).
         Mounting /mnt...
         Mounting /var/volatile...
[  OK  ] Mounted /mnt.
[  OK  ] Mounted /var/volatile.
         Starting Load/Save Random Seed...
[  OK  ] Started Load/Save Random Seed.
[  OK  ] Started udev Wait for Complete Device Initialization.
[  OK  ] Started Hardware RNG Entropy Gatherer Daemon.
[  OK  ] Started Spirit early boot setup.
[  OK  ] Reached target Local File Systems.
         Starting Rebuild Dynamic Linker Cache...
         Starting SELinux init service loading...
         Starting Create Volatile Files and Directories...
[  OK  ] Started SELinux init service loading.
[  OK  ] Started Rebuild Dynamic Linker Cache.
[  OK  ] Started Create Volatile Files and Directories.
         Starting Security Auditing Service...
         Starting Run pending postinsts...
         Starting Rebuild Journal Catalog...
[  OK  ] Started Security Auditing Service.
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Started Run pending postinsts.
[  OK  ] Started Rebuild Journal Catalog.
         Starting Update is Completed...
[  OK  ] Started Update is Completed.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Periodic rotation of log files.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target Timers.
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Listening on Libvirt local socket.
[  OK  ] Listening on Libvirt admin socket.
[  OK  ] Listening on Libvirt local read-only socket.
[  OK  ] Listening on Tftp Server Activation Socket.
[  OK  ] Listening on Virtual machine lock manager socket.
[  OK  ] Listening on Virtual machine log manager socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
         Starting Avahi mDNS/DNS-SD Stack...
[  OK  ] Started Periodic Command Scheduler.
[  OK  ] Started D-Bus System Message Bus.
[  OK  ] Started diskmon.
         Starting Ethernet Bridge Filtering Tables...
         Starting IPv6 Packet Filtering Framework...
         Starting IPv4 Packet Filtering Framework...
         Starting Cisco kdump service...
         Starting Lighttpd Daemon...
[  OK  ] Started Machine Check Exception Logging Daemon.
[  OK  ] Started Cisco Process Monitor Service.
         Starting Rollback uncommit… config change transactions...
         Starting System Logging Service...
[  OK  ] Started Self Monitoring an…ing Technology (SMART) Daemon.
[  OK  ] Started Cisco Serial Getty Service.
[  OK  ] Reached target Cisco custom login prompts.
         Starting Login Service...
         Starting Virtual Machine a…tainer Registration Service...
         Starting XR installation service...
         Starting OpenSSH Key Generation...
[  OK  ] Started Ethernet Bridge Filtering Tables.
[  OK  ] Started IPv6 Packet Filtering Framework.
[  OK  ] Started IPv4 Packet Filtering Framework.
[  OK  ] Started Rollback uncommitt…rk config change transactions.
[  OK  ] Started XR installation service.
[  OK  ] Reached target Network (Pre).
         Starting Network Service...
         Starting Rotate log files...
[  OK  ] Started Lighttpd Daemon.
[  OK  ] Started System Logging Service.
[  OK  ] Started Network Service.
[  OK  ] Started Rotate log files.
[  OK  ] Started Login Service.
[  OK  ] Started Virtual Machine an…ontainer Registration Service.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
[  OK  ] Reached target Network.
         Starting Create Network Namespaces...
         Starting Virtualization daemon...
[  OK  ] Started Respond to IPv6 Node Information Queries.
[  OK  ] Started Network Router Discovery Daemon.
         Starting Permit User Sessions...
[  OK  ] Started Xinetd A Powerful Replacement For Inetd.
[  OK  ] Started Permit User Sessions.
[  OK  ] Started OpenSSH Key Generation.
         Starting OpenSSH server daemon...
[  OK  ] Started OpenSSH server daemon.
[  OK  ] Started Create Network Namespaces.
[  OK  ] Started Virtualization daemon.
[  OK  ] Started Host Authentication service.
         Starting Suspend/Resume Running libvirt Guests...
[  OK  ] Started Suspend/Resume Running libvirt Guests.
[  OK  ] Reached target Multi-User System.
[  OK  ] Started Network bridge for containers.
[  OK  ] Started Spirit service to kickstart XR processes.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
[  OK  ] Started Launch console access for calvados aux port.
[  OK  ] Started Launch console access for calvados console port.
[  OK  ] Started Launch console access for XR console port.
[  OK  ] Created slice system-cisco\x2dserial\x2dgetty.slice.
[  OK  ] Started Serial Getty on ttyS1.
[  OK  ] Started Calvados service.
[  OK  ] Started Hushd service.
[  OK  ] Started Oc_host service.

Linux Distro for XR 11.1.2 ios /dev/ttyS0


[  OK  ] Stopped Oc_host service.
[  OK  ] Started Oc_host service.
[  OK  ] Stopped Oc_host service.
[  OK  ] Started Oc_host service.
[  OK  ] Stopped Oc_host service.
[  OK  ] Started Oc_host service.
[  OK  ] Started Cisco kdump service.
[  OK  ] Reached target XR installation and startup.
[  OK  ] Stopped Oc_host service.
[  OK  ] Started Oc_host service.
Telnet escape character is '^Q'.
Trying ::1...
Connection failed: Connection refused
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^Q'.
^@^@^@^@



systemd 244.5+ running in system mode. (+PAM +AUDIT +SELINUX +IMA -APPARMOR -SMACK +SYSVINIT +UTMP -LIBCRYPTSETUP -GCRYPT -GNUTLS +ACL +XZ -LZ4 -SECCOMP +BLKID -ELFUTILS +KMOD -IDN2 -IDN -PCRE2 default-hierarchy=hybrid)
Detected virtualization lxc-libvirt.
Detected architecture x86-64.
Failed to create symlink /sys/fs/cgroup/cpu: File exists
Failed to create symlink /sys/fs/cgroup/cpuacct: File exists
Failed to create symlink /sys/fs/cgroup/net_cls: File exists

Welcome to Linux Distro for XR 11.1.2 (dunfell)!

Set hostname to <ios>.
Initializing machine ID from container UUID.
Couldn't move remaining userspace processes, ignoring: Input/output error
user.slice: unit configures an IP firewall, but the local system does not support BPF/cgroup firewalling.
(This warning is only shown for the first unit using IP firewalling.)
[  OK  ] Created slice User and Session Slice.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[  OK  ] Reached target Paths.
[  OK  ] Reached target Remote File Systems.
[  OK  ] Reached target Slices.
[  OK  ] Reached target Swap.
[  OK  ] Reached target Libvirt guests shutdown.
[  OK  ] Listening on Device-mapper event daemon FIFOs.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Reached target RPC Port Mapper.
[  OK  ] Listening on Syslog Socket.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket.
[  OK  ] Listening on Network Service Netlink Socket.
         Mounting Huge Pages File System...
         Mounting POSIX Message Queue File System...
         Mounting /proc/cmdline...
         Mounting Kernel Debug File System...
         Mounting Temporary Directory (/tmp)...
         Starting Availability of block devices...
         Starting Monitoring of LVM…meventd or progress polling...
         Starting RPC Bind...
         Starting SELinux autorelabel service loading...
         Starting SELinux init for /dev service loading...
         Starting Spirit early boot setup...
         Starting Journal Service...
         Mounting FUSE Control File System...
         Starting Remount Root and Kernel File Systems...
[  OK  ] Started Hardware RNG Entropy Gatherer Daemon.
[  OK  ] Mounted Huge Pages File System.
[  OK  ] Mounted POSIX Message Queue File System.
[  OK  ] Mounted /proc/cmdline.
[  OK  ] Mounted Kernel Debug File System.
[  OK  ] Mounted Temporary Directory (/tmp).
[  OK  ] Started Availability of block devices.
selinux-labeldev.service: Succeeded.
[  OK  ] Started SELinux init for /dev service loading.
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Started Remount Root and Kernel File Systems.
         Starting Create System Users...
[  OK  ] Started Journal Service.
         Starting Flush Journal to Persistent Storage...
[  OK  ] Started RPC Bind.
[  OK  ] Started SELinux autorelabel service loading.
[  OK  ] Started Flush Journal to Persistent Storage.
[  OK  ] Started Create System Users.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Started Monitoring of LVM2… dmeventd or progress polling.
[  OK  ] Started Create Static Device Nodes in /dev.
[  OK  ] Reached target Local File Systems (Pre).
         Mounting /mnt...
         Mounting /var/volatile...
[  OK  ] Mounted /mnt.
[  OK  ] Mounted /var/volatile.
[  OK  ] Started Spirit early boot setup.
[  OK  ] Reached target Local File Systems.
         Starting Rebuild Dynamic Linker Cache...
         Starting SELinux init service loading...
         Starting Create Volatile Files and Directories...
[  OK  ] Started Create Volatile Files and Directories.
         Starting Security Auditing Service...
         Starting Rebuild Journal Catalog...
[  OK  ] Started Security Auditing Service.
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Started Rebuild Journal Catalog.
[  OK  ] Started SELinux init service loading.
[  OK  ] Started Rebuild Dynamic Linker Cache.
         Starting Run pending postinsts...
         Starting Update is Completed...
[  OK  ] Started Update is Completed.
[  OK  ] Started Run pending postinsts.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Periodic rotation of log files.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target Timers.
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[FAILED] Failed to listen on Libvirt local socket.
See 'systemctl status libvirtd.socket' for details.
[DEPEND] Dependency failed for Libvirt local read-only socket.
[  OK  ] Listening on Tftp Server Activation Socket.
[  OK  ] Listening on Virtual machine lock manager socket.
[  OK  ] Listening on Virtual machine log manager socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
         Starting Avahi mDNS/DNS-SD Stack...
[  OK  ] Started Periodic Command Scheduler.
[  OK  ] Started D-Bus System Message Bus.
         Starting Ethernet Bridge Filtering Tables...
         Starting IPv6 Packet Filtering Framework...
         Starting IPv4 Packet Filtering Framework...
         Starting Cisco kdump service...
         Starting Lighttpd Daemon...
[  OK  ] Started Machine Check Exception Logging Daemon.
[  OK  ] Started Cisco Process Monitor Service.
         Starting Rollback uncommit… config change transactions...
         Starting System Logging Service...
[  OK  ] Started Cisco Serial Getty Service.
[  OK  ] Reached target Cisco custom login prompts.
         Starting Login Service...
[  OK  ] Started XR pre-shutdown service.
         Starting xr_sysctl.service...
         Starting XR installation service...
         Starting OpenSSH Key Generation...
[  OK  ] Started Ethernet Bridge Filtering Tables.
         Starting Rotate log files...
[  OK  ] Started IPv6 Packet Filtering Framework.
[  OK  ] Started IPv4 Packet Filtering Framework.
[  OK  ] Reached target Network (Pre).
         Starting Network Service...
[  OK  ] Started System Logging Service.
[  OK  ] Started Lighttpd Daemon.
[  OK  ] Started Network Service.
[  OK  ] Started Rollback uncommitt…rk config change transactions.
[  OK  ] Started xr_sysctl.service.
[  OK  ] Started XR installation service.
[  OK  ] Started Rotate log files.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
[  OK  ] Reached target Network.
         Starting install-apps.service...
         Starting Create Network Namespaces...
         Starting Suspend/Resume Running libvirt Guests...
[  OK  ] Started Respond to IPv6 Node Information Queries.
[  OK  ] Started Network Router Discovery Daemon.
         Starting Permit User Sessions...
[  OK  ] Started Xinetd A Powerful Replacement For Inetd.
         Starting xrv9k_xr.sysinit.service...
[  OK  ] Started install-apps.service.
[  OK  ] Started Create Network Namespaces.
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Suspend/Resume Running libvirt Guests.
[  OK  ] Started Login Service.
[FAILED] Failed to start xrv9k_xr.sysinit.service.
See 'systemctl status xrv9k_xr.sysinit.service' for details.
[  OK  ] Started OpenSSH Key Generation.
[  OK  ] Stopped Machine Check Exception Logging Daemon.
[  OK  ] Started Machine Check Exception Logging Daemon.
         Starting OpenSSH server daemon...
[  OK  ] Started OpenSSH server daemon.
[  OK  ] Reached target Multi-User System.
[  OK  ] Started Spirit service to kickstart XR processes.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
[  OK  ] Stopped Machine Check Exception Logging Daemon.
[  OK  ] Started Machine Check Exception Logging Daemon.
[  OK  ] Started Cisco kdump service.
[  OK  ] Reached target XR installation and startup.
[  OK  ] Stopped Machine Check Exception Logging Daemon.
[  OK  ] Started Machine Check Exception Logging Daemon.


ios con0/RP0/CPU0 is now available





Press RETURN to get started.

0/RP0/ADMIN0:Jan  9 13:52:46.000 UTC: inst_agent[1842]: %INFRA-INSTAGENT-4-XR_PART_PREP_REQ : Received SDR/XR partition request. Looking for available matching partition. If not found, new one will be created after copying relevant image and RPMs
<mark>0/RP0/ADMIN0:Jan  9 13:52:50.000 UTC: inst_agent[1842]: %INFRA-INSTAGENT-4-XR_PART_PREP_IMG : SDR/XR image baking in progress
0/RP0/ADMIN0:Jan  9 13:53:39.000 UTC: inst_agent[1842]: %INFRA-INSTAGENT-4-XR_PART_PREP_RESP : SDR/XR partition preparation completed successfully</mark>




This product contains cryptographic features and is subject to United
States and local country laws governing import, export, transfer and
use. Delivery of Cisco cryptographic products does not imply third-party
authority to import, export, distribute or use encryption. Importers,
exporters, distributors and users are responsible for compliance with
U.S. and local country laws. By using this product you agree to comply
with applicable laws and regulations. If you are unable to comply with
U.S. and local laws, return this product immediately.

A summary of U.S. laws governing Cisco cryptographic products may be
found at:
http://www.cisco.com/wwl/export/crypto/tool/stqrg.html

If you require further assistance please contact us by sending email to
export@cisco.com.



RP/0/RP0/CPU0:Jan  9 13:53:45.636 UTC: pyztp2[208]: %INFRA-ZTP-4-START : ZTP has started. Interfaces might be brought up if they are shutdown
0/RP0/ADMIN0:Jan  9 13:53:51.000 UTC: vm_manager[1897]: %INFRA-VM_MANAGER-4-INFO : Info: vm_manager started VM default-sdr--2
RP/0/RP0/CPU0:Jan  9 13:54:11.717 UTC: smartlicserver[390]: %LICENSE-SMART_LIC-3-COMM_FAILED : Communications failure with the Cisco Smart License Utility (CSLU) : Unable to resolve server hostname/domain name
RP/0/RP0/CPU0:Jan  9 13:54:17.108 UTC: pyztp2[208]: %INFRA-ZTP-3-FETCHING_FAILED : Fetching failed. ZTP will retry.
</code>
</pre>
</div>

Once ready, system will request user creation after first boot:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
!!!!!!!!!!!!!!!!!!!! NO root-system username is configured. Need to configure root-system username. !!!!!!!!!!!!!!!!!!!!

         --- Administrative User Dialog ---


  Enter root-system username: admin
  Enter secret:
  Enter secret again:
Use the 'configure' command to modify this configuration.
User Access Verification

Username: admin
Password:


RP/0/RP0/CPU0:ios
</code>
</pre>
</div>

_Et voila._

The system is ready :

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:RR-1#sh platform
Thu Jan  9 21:54:37.429 UTC
Node              Type                       State             Config state
--------------------------------------------------------------------------------
0/0/CPU0          R-IOSXRV9000-LC-A          <mark>IOS XR RUN</mark>        NSHUT
0/RP0/CPU0        R-IOSXRV9000-RP-A(Active)  <mark>IOS XR RUN</mark>        NSHUT
0/FT0             XRV9K-GENERIC-FT           OPERATIONAL       NSHUT
0/PT0             XRV9K-GENERIC-PT           OPERATIONAL       NSHUT
RP/0/RP0/CPU0:RR-1#
</code>
</pre>
</div>

And runs IOS XR 24.4.1 :
<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:RR-1#show version
Thu Jan  9 21:55:13.906 UTC
Cisco IOS XR Software, Version 24.4.1
Copyright (c) 2013-2024 by Cisco Systems, Inc.

Build Information:
 Built By     : swtools
 Built On     : Tue Dec 17 07:14:03 PST 2024
 Built Host   : iox-ucs-078
 Workspace    : /auto/srcarchive10/prod/24.4.1/xrv9k/ws
 <mark>Version      : 24.4.1</mark>
 Location     : /opt/cisco/XR/packages/
 Label        : 24.4.1

cisco IOS-XRv 9000 () processor
System uptime is 7 hours 2 minutes

RP/0/RP0/CPU0:RR-1#sh install active summary
Thu Jan  9 21:55:19.429 UTC
Label : 24.4.1

    Active Packages: 1
        xrv9k-xr-24.4.1 version=24.4.1 [Boot image]

RP/0/RP0/CPU0:RR-1#
</code>
</pre>
</div>

# Video
The whole process is illustrated in following video:

# Conclusion
This document covered reimage procedure which can be used to stage or recover an IOS XRv 9000 appliance based on Cisco UCS M7 server. While IOS XR 24.4.1 was used to illustrate it, the method will be similar for upcoming software releases.