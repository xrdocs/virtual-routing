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


## Selecting the Host Machine

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










Part-4 of the XRd tutorials Series: [here]({{base_path}}/2022-08-23-xrd-with-docker-control-plane-and-vrouter).
