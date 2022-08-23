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
* Linux cgroups version 1 (unified hierarchy cgroups not yet supported)

Further, each XRd vRouter instance running on the host must have the following resources allocated to it:

* 2 isolated* CPUs
* 5GiB RAM
* 3GiB additional hugepage RAM
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

Intel i350 Quad Port 1Gb Adapter
Intel Dual Port 10 GbE Ethernet X520 Server Adapter
Intel 4 port 10GE Fortville
Cisco UCS Virtual Interface Card (VIC) 1225

In addition to this virtual NICs such as e1000 or VMXNET3 are supported albeit with much lower throughput than an interface passthrough.



## Selecting the Host Machine

In this tutorial, we select **Ubuntu 20.04** as the underlying distribution for the host machine.
This meets most of the requirements when it comes to the kernel version and the user-space libraries (like docker) that are available for this distribution.

The host machine selected is a virtual machine hosted on VMWare ESXI. 

<p class="notice--info">
**Note**: The host machine can be a bare-metal server or any other hypervisor such as VMWare fusion, KVM etc. that support exposing IOMMU to the guest OS.  
<br/>
For e.g., in case of VMWare ESXI, select "Expose IOMMU to guest OS" under the CPU section for the virtual machine as shown below:
  
<img src="{{base_path}}/images/" alt="Open/R integration with IOS-XR- current design">
<p/>

The specs for the host machine selected are:
* 8 CPUs
* 30GiB RAM














Part-4 of the XRd tutorials Series: [here]({{base_path}}/2022-08-23-xrd-with-docker-control-plane-and-vrouter).
