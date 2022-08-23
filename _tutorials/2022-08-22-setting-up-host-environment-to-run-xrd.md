---
published: true
date: '2022-08-22 18:59 +0530'
title: 'Setting up the Host Environment to run XRd '
author: Akshat Sharma
excerpt: >-
  Learn how to set up the host environment to successfully launch Control-Plane
  and vRouter XRd images in your setup.
position: hidden
tags:
  - iosxr
  - cisco
  - xrd
  - xrv9k
  - vm
  - container
  - docker
  - kubernetes
  - virtual
  - xrd-tutorial-series
---

{% include base_path %}
{% include toc %}

This is Part-3 of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series). Skip to Part-4 [here]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter). 


## Introduction

XRd is available today as a docker image tarball to be deployed in containerized network deployments. This implies the host machine on which the container is orchestrated must supply the kernel and the necessary host devices and settings (pci passthrough, HugePages etc.) to allow XRd to run smoothly. The host machine could therefore be a baremetal server or a virtual machine as long as it meets certain minimum requirements. What are these requirements ?


## XRd Host Machine Requirements

The requirements for each XRd form factor are given below:

### XRd Control-Plane 

For XRd Control-Plane containers, the host must meet the following minimum requirements:

* An x86_64 CPU with at least 2 CPU cores
* 4GiB RAM
* Linux kernel version 4+
  * With the 'dummy' and 'nf_tables' kernel modules installed
* Linux cgroups version 1 (unified hierarchy cgroups not yet supported)

Further, each XRd Control Plane instance running on the host must have the following resources allocated to it:

* 1 CPU
* 2GiB RAM
* 2000 inotify user instances and watches


### XRd vRouter

For XRd vRouter, the minimum requirements for the host entail:

* An x86_64 CPU with:
  * At least 4 CPU cores
  * Support for the SSSE3, SSE4.1 and SSE4.2 instruction sets
* Linux kernel version 4+, with the following modules installed:
  * dummy
  * nf_tables
  * vfio-pci or igb_uio
  The host must have installed and enabled the vfio-pci and/or igb_uio kernel module(s). When 
  using vfio-pci it is strongly recommended to use IOMMU to allow safe pass-through of PCI 
  devices.
  Additionally, the user must ensure /dev/vfio is mounted in the XRd container.

* Linux cgroups version 1 (unified hierarchy cgroups not yet supported)
* Security Modules: XRd vRouter does not currently support enforcing security modules such as 
  AppArmor or SELinux and this must either be disabled on the host or when starting the container.

Further, each XRd vRouter instance running on the host must have the following resources allocated to it:

* 2 isolated CPUs
  There must be at least 2 CPUs available to XRd vRouter. By default XRd vRouter uses a single CPU 
  core for the dataplane packet thread (the last CPU it is allowed to use). In order for the 
  dataplane to perform properly it needs dedicated use of this physical CPU core. It is up to the 
  user to ensure other workloads are not using this CPU.  
    
  This is especially important when launching multiple XRd vRouter instances, as by default they 
  will all use the same CPU. In this scenario one can use the runtime to control the cpuset each 
  container is allowed to use (e.g. --cpuset-cpus with docker) and/or the XRd environment variable  
* 5GiB RAM
* 3GiB additional hugepage RAM  
  Hugepage support must be enabled, with 2MiB or 1GiB hugepage size supported. There must be 3GiB 
  of available hugepages per XRd vRouter instance.
* 2000 inotify user instances and watches


### Docker Version

Docker version 18+ is required, along with permissions to run Docker containers


### Host Distribution

Host distributions which have been tested and are supported for use with XRd are:

* Ubuntu 18.04/20.04
* CentOS 8.2

**Note**: At times there can be incompatibilities with the default LSM policies (e.g. SELinux under RedHat), and these may need to be disabled to get going
{: .notice--info}

DO NOT USE RHEL/CENTOS 8.3
There is a known issue with the kernel version used in RHEL/CentOS kernel 8.3 that has been acknowledged by RedHat and fixed in the kernel used for 8.4. Unfortunately, it is not expected that the fix will be backported to 8.3, so this version must be avoided for running all XRd platforms.
{: .notice--danger}


### Supported NICs

The following physical NICs are supported for interface passthrough to XRd vRouter:

* Intel i350 Quad Port 1Gb Adapter
* Intel Dual Port 10 GbE Ethernet X520 Server Adapter
* Intel 4 port 10GE Fortville
* Cisco UCS Virtual Interface Card (VIC) 1225

In addition to this virtual NICs such as e1000 or VMXNET3 are supported albeit with much lower throughput than an interface passthrough.



## Selecting the Host Machine

In this tutorial, we select **Ubuntu 20.04** as the underlying distribution for the host machine.
This meets most of the requirements when it comes to the kernel version and the user-space libraries (like docker) that are available for this distribution.

The host machine selected is a virtual machine hosted on VMWare ESXI. 

<p class="notice--info">
<b>Note</b>: The host machine can be a bare-metal server or any other hypervisor such as VMWare fusion, KVM etc. that support exposing IOMMU to the guest OS.  
<br/>
For e.g., in case of VMWare ESXI, select "Expose IOMMU to guest OS" under the CPU section for the virtual machine as shown below:
</p>
<p class="notice--primary"> 
<img src="{{base_path}}/images/iommu_enable_esxi.png" alt="IOMMU enable guest OS ESXI"></p>



The specs for the host machine selected are:
* 8 CPUs
* 30GiB RAM

With these resources, (allocating about 5GiB RAM along with 3GiB additional HugePages RAM and minimum 2 CPUs for each XRd vRouter), we should be able to easily handle about 3 XRd vRouters and around 7-8 XRd Control-Plane routers.
We'll use this as the upper limit for the number of routers in each topology we attempt in subsequent tutorials.




## Configuring the Host Machine

### Host-Check script

In the [xrd-tools tutorial]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd), we introduced the "host-check" script that is published to github:  

>[https://github.com/ios-xr/xrd-tools/blob/main/scripts/host-check](https://github.com/ios-xr/xrd-tools/blob/main/scripts/host-check)


This script is an extremely handy tool to determine the suitability of the Host machine for either XRd platform - control-plane or vRouter.



### Running the Host-Check Script

For our freshly booted Ubuntu 20.04 Host machine, let's run the `host-check` script to check the initial state of the host before we make corrections as needed:


* First, for the XRd Control-Plane platform:
  
  ```bash
  cisco@xrdcisco:~/xrd-tools/scripts$ ./host-check --platform xrd-control-plane
  ==============================
  Platform checks - xrd-control-plane 
  ==============================
  PASS -- CPU architecture (x86_64)
  PASS -- CPU cores (8)
  PASS -- Kernel version (5.4)
  PASS -- Base kernel modules
          Installed module(s): dummy, nf_tables
  PASS -- Cgroups version (v1)
  PASS -- systemd mounts
          /sys/fs/cgroup and /sys/fs/cgroup/systemd mounted correctly.
  FAIL -- Inotify max user instances
          The kernel parameter fs.inotify.max_user_instances is set to 128 but
          should be at least 4000 (sufficient for a single instance) - the
          recommended value is 64000.
          This can be addressed by adding 'fs.inotify.max_user_instances=64000'
          to /etc/sysctl.conf or in a dedicated conf file under /etc/sysctl.d/.
          For a temporary fix, run:
            sysctl -w fs.inotify.max_user_instances=64000
  WARN -- Inotify max user watches
          The kernel parameter fs.inotify.max_user_watches is set to 8192 -
          this is expected to be sufficient for 2 XRd instance(s).
          The recommended value is 64000.
          This can be addressed by adding 'fs.inotify.max_user_watches=64000'
          to /etc/sysctl.conf or in a dedicated conf file under /etc/sysctl.d/.
          For a temporary fix, run:
            sysctl -w fs.inotify.max_user_watches=64000
  INFO -- Core pattern (core files managed by the host)
  PASS -- ASLR (full randomization)
  INFO -- Linux Security Modules
          AppArmor is enabled. XRd is currently unable to run with the
          default docker profile, but can be run with
          '--security-opt apparmor=unconfined' or equivalent.
  PASS -- RAM
          Available RAM is 28.7 GiB.
          This is estimated to be sufficient for 14 XRd instance(s), although memory
          usage depends on the running configuration.
          Note that any swap that may be available is not included.

  ==================================================================  
  !! Host NOT set up correctly for xrd-control-plane !!
  ==================================================================
  cisco@xrdcisco:~/xrd-tools/scripts$ 
  
  ```

* Secondly, for the XRd vRouter Platform:

  ```bash
  cisco@xrdcisco:~/xrd-tools/scripts$ ./host-check --platform xrd-vrouter
  ==============================
  Platform checks - xrd-vrouter
  ==============================
  PASS -- CPU architecture (x86_64)  
  PASS -- CPU cores (8)
  PASS -- Kernel version (5.4)
  PASS -- Base kernel modules
          Installed module(s): dummy, nf_tables
  PASS -- Cgroups version (v1)
  PASS -- systemd mounts
          /sys/fs/cgroup and /sys/fs/cgroup/systemd mounted correctly.
  FAIL -- Inotify max user instances
          The kernel parameter fs.inotify.max_user_instances is set to 128 but
          should be at least 4000 (sufficient for a single instance) - the
          recommended value is 64000.
          This can be addressed by adding 'fs.inotify.max_user_instances=64000'
          to /etc/sysctl.conf or in a dedicated conf file under /etc/sysctl.d/.
          For a temporary fix, run:
            sysctl -w fs.inotify.max_user_instances=64000
  WARN -- Inotify max user watches
          The kernel parameter fs.inotify.max_user_watches is set to 8192 -
          this is expected to be sufficient for 2 XRd instance(s).
          The recommended value is 64000.
          This can be addressed by adding 'fs.inotify.max_user_watches=64000'
          to /etc/sysctl.conf or in a dedicated conf file under /etc/sysctl.d/.
          For a temporary fix, run:
            sysctl -w fs.inotify.max_user_watches=64000
  INFO -- Core pattern (core files managed by the host)
  PASS -- ASLR (full randomization)
  INFO -- Linux Security Modules
          AppArmor is enabled. XRd is currently unable to run with the
          default docker profile, but can be run with
          '--security-opt apparmor=unconfined' or equivalent.
  PASS -- CPU extensions (sse4_1, sse4_2, ssse3)
  PASS -- RAM
          Available RAM is 28.7 GiB.
          This is estimated to be sufficient for 5 XRd instance(s), although memory
          usage depends on the running configuration.
          Note that any swap that may be available is not included.
  FAIL -- Hugepages
          Hugepages are not enabled. These are required for XRd to function correctly.
          To enable hugepages, see the instructions at:
          https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt.
  PASS -- Interface kernel driver (vfio-pci loaded)
  FAIL -- IOMMU
          The kernel module vfio-pci cannot be used, as IOMMU is not enabled.
          IOMMU is recommended for security when using the vfio-pci kernel driver.
  PASS -- Shared memory pages max size (17179869184.0 GiB)
  
  ==================================================================
  !! Host NOT set up correctly for xrd-vrouter !!
  ==================================================================
  cisco@xrdcisco:~/xrd-tools/scripts$ 

  ```
  
**Note**: The host-check script does not attempt to fix any problems, although it tries to give hints about how they can be addressed. Most of the solutions can be found fairly easily by searching the internet.
{: .notice--warning}

### Making Suggested Corrections to the Host Machine

**Note**: The changes beind made in this section are relevant to the distribution (Ubuntu 20.04) selected for the host machine in this example. If you have selected some other distribution, then look for **equivalent** solutions for the distribution you're using by searching online.
{: .notice--danger}


Based on the output of the `host-check` script run for each platform type above, we will focus on all the `FAIL` and `WARN` scenarios and correct them as suggested.  


#### Inotify max user watches and Inotify max user instances settings

These settings are common to both XRd Control-Plane and XRd vRouter platform types.

Fix the max_user_watches and max_user_instances settings by adding the following two lines to `/etc/sysctl.conf`:

```bash
cisco@xrdcisco:~$ cat  /etc/sysctl.conf  | tail -2
fs.inotify.max_user_instances=64000
fs.inotify.max_user_watches=64000
cisco@xrdcisco:~$ 

```

#### IOMMU and HugePages Settings

Next, for XRd vRouter to work, enable iommu and Hugepages for the Host machine. HugePages of size 1GiB must be enabled with a total of 3GiB of available HugePages RAM for **each** XRd vRouter.

To enable iommu appropriately, determine the Host machine type first (intel vs AMD):

```
cisco@xrdcisco:~$ lscpu | grep "Vendor ID:"
Vendor ID:                       GenuineIntel
cisco@xrdcisco:~$ 
```

**Note**: `GenuineIntel` indicates an intel machine while `AuthenticAMD` will indicate an AMD machine.  
{: .notice--info}  

Add `intel_iommu=on iommu=pt` to GRUB_CMDLINE_LINUX in /etc/default/grub (relevant to the Ubuntu distribution in use here) to enable IOMMU for any PCI passthrough interfaces we would use later.  

Secondly, to set a total HugePage allocation of 9GiB (3GiB per XRd vRouter with a max of 3 XRd vRouters that we're looking to support here), add `default_hugepagesz=1G hugepagesz=1G hugepages=9` to GRUB_CMDLINE_LINUX in /etc/default/grub as well.

These changes are highlighted below

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">
cisco@xrdcisco:~$ <mark>cat /etc/default/grub</mark>
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity"
GRUB_CMDLINE_LINUX=<mark>"intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G hugepages=9"</mark>

# Uncomment to enable BadRAM filtering, modify to suit your needs
# This works with Linux (no patch required) and with any kernel that obtains
# the memory map information from GRUB (GNU Mach, kernel of FreeBSD ...)
#GRUB_BADRAM="0x01234567,0xfefefefe,0x89abcdef,0xefefefef"

# Uncomment to disable graphical terminal (grub-pc only)
#GRUB_TERMINAL=console

# The resolution used on graphical terminal
# note that you can use only modes which your graphic card supports via VBE
# you can see them in real GRUB with the command `vbeinfo'
#GRUB_GFXMODE=640x480

# Uncomment if you don't want GRUB to pass "root=UUID=xxx" parameter to Linux
#GRUB_DISABLE_LINUX_UUID=true

# Uncomment to disable generation of recovery mode menu entries
#GRUB_DISABLE_RECOVERY="true"

# Uncomment to get a beep at grub start
#GRUB_INIT_TUNE="480 440 1"
cisco@xrdcisco:~$ 
</code>
</pre>
</div>


Finally, update the grub configuration by running `sudo update-grub`:

```
cisco@xrdcisco:~/xrd-tools/scripts$ sudo update-grub
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.4.0-81-generic
Found initrd image: /boot/initrd.img-5.4.0-81-generic
done
cisco@xrdcisco:~/xrd-tools/scripts$ 
```   

#### Restart the Host Machine

**Very Important**: Make sure you restart the host machine post any grub config changes to make sure they take effect.  
{: .notice--warning}  

```
cisco@xrdcisco:~/xrd-tools/scripts$ sudo shutdown -r now
Connection to x.x.x.x closed by remote host.
Connection to x.x.x.x closed.

```

### Re-Run Host-Check script post Corrections

Once the host machine is back up again, re-run the `host-check` script to make sure it indicates that the host machine is fit for both XRd Control-Plane and XRd vRouter platforms.  


For XRd Control-Plane Platform:

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">
cisco@xrdcisco:~/xrd-tools/scripts$ ./host-check --platform xrd-control-plane
==============================
Platform checks - xrd-control-plane
==============================
PASS -- CPU architecture (x86_64)
PASS -- CPU cores (8)
PASS -- Kernel version (5.4)
PASS -- Base kernel modules
        Installed module(s): dummy, nf_tables
PASS -- Cgroups version (v1)
PASS -- systemd mounts
        /sys/fs/cgroup and /sys/fs/cgroup/systemd mounted correctly.
<mark>PASS -- Inotify max user instances
        64000 - this is expected to be sufficient for 16 XRd instance(s).
PASS -- Inotify max user watches
        64000 - this is expected to be sufficient for 16 XRd instance(s).</mark>
INFO -- Core pattern (core files managed by the host)
PASS -- ASLR (full randomization)
INFO -- Linux Security Modules
        AppArmor is enabled. XRd is currently unable to run with the
        default docker profile, but can be run with
        '--security-opt apparmor=unconfined' or equivalent.
PASS -- RAM
        Available RAM is 19.8 GiB.
        This is estimated to be sufficient for 9 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.

==================================================================
<mark>Host environment set up correctly for xrd-control-plane</mark>
==================================================================
cisco@xrdcisco:~/xrd-tools/scripts$ 
</code>
</pre>
</div>  

Similarly, for the XRd vRouter platform:  

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">
cisco@xrdcisco:~/xrd-tools/scripts$ ./host-check --platform xrd-vrouter
==============================
Platform checks - xrd-vrouter
==============================
PASS -- CPU architecture (x86_64)
PASS -- CPU cores (8)
PASS -- Kernel version (5.4)
PASS -- Base kernel modules
        Installed module(s): dummy, nf_tables
PASS -- Cgroups version (v1)
PASS -- systemd mounts
        /sys/fs/cgroup and /sys/fs/cgroup/systemd mounted correctly.
PASS -- Inotify max user instances
        64000 - this is expected to be sufficient for 16 XRd instance(s).
PASS -- Inotify max user watches
        64000 - this is expected to be sufficient for 16 XRd instance(s).
INFO -- Core pattern (core files managed by the host)
PASS -- ASLR (full randomization)
INFO -- Linux Security Modules
        AppArmor is enabled. XRd is currently unable to run with the
        default docker profile, but can be run with
        '--security-opt apparmor=unconfined' or equivalent.
PASS -- CPU extensions (sse4_1, sse4_2, ssse3)
PASS -- RAM
        Available RAM is 19.8 GiB.
        This is estimated to be sufficient for 3 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.
<mark>PASS -- Hugepages (9 x 1GiB)
PASS -- Interface kernel driver (vfio-pci loaded)
PASS -- IOMMU
        IOMMU enabled for vfio-pci with the following PCI device(s):
        ens33 (0000:02:01.0), ens160 (0000:03:00.0), ens192 (0000:0b:00.0),
        ens224 (0000:13:00.0), ens256 (0000:1b:00.0)</mark>
PASS -- Shared memory pages max size (17179869184.0 GiB)

==================================================================
<mark>Host environment set up correctly for xrd-vrouter </mark>
==================================================================
cisco@xrdcisco:~/xrd-tools/scripts$

</code>
</pre>
</div>  




And that's it for setting up the host environment. The `host-check` script makes it super-easy to keep track of the settings to take care of when running XRd Control-Plane or vRouter variants. Next up - let's try spinning up XRd using docker!  
{: .notice--success}












Part-4 of the XRd tutorials Series: [here]({{base_path}}/2022-08-23-xrd-with-docker-control-plane-and-vrouter).
