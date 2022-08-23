---
published: true
date: '2022-08-22 11:27 +0530'
title: 'XRd images: Where can one get them?'
author: Akshat Sharma
position: hidden
tags:
  - iosxr
  - cisco
  - virtual
  - xrd
  - xrv9k
  - docker
  - container
  - vm
  - kubernetes
excerpt: Learn how to download the latest XRd images and set them up for further use.
---

{% include base_path %}
{% include toc %}

## Introduction
This tutorial will help the user learn how to gain access to the latest official XRd images from Cisco and set them up for further use with docker, kubernetes and related tools. In subsequent tutorials, we will dive deeper into the environments and tools needed to run XRd locally and discuss production deployment strategies and techniques.

In our earlier [blog]({{base_path}}/blogs/), we introduced XRd as the latest Virtual Platform offering for the Service-Provider market with its initial focus on Containerized network deployments in on-prem data centers and the public cloud. The obvious question to start with is: How do I get my hands on the image? Read on to find out.


## XRd Images Form Factors

XRd has two form factors:

1. **XRd Control Plane:** As the name suggests, this version of XRd is meant for control-plane only use cases that do not require high-throughput, traffic forwarding capabilities. These use cases include but are not limited to - vRR (virtual Route Reflector), SR-PCE (in tandem with a policy controller such as Cisco Network Controller - CNC).

2. **XRd vRouter:** This version of XRd contains a fully-featured and performant software dataplane in addition to the fully-functional XR control-plane, and may be suitable for deployments that require data traffic forwarding such as vPE (virtual Provider Edge), vCSR (virtual Cell Site Router) and Cloud Router (Public Cloud based vRouter).

<p class="notice--info">
  <b>Note:</b> The choice between the two variants is hinged on whether a user needs a software data-plane and associated features with traffic-forwarding capabilities or not. 
<br/><br/>
While XRd vRouter could also satisfy the Control-Plane only use cases, it requires more resources (cpu, memory, Huge Pages additional RAM) and specific enablements (IOMMU enabled host devices such as PCI passthrough or e1000/VMXNET3 interfaces). The control-plane only XRd image is comparitively less resource intensive and easier to deploy in existing containerized deployments.
</p>


## Download Images from CCO

Both these variants of XRd are available on CCO ([https://software.cisco.com/downloads](https://software.cisco.com/downloads)) as tarballs that contain the corresponding docker image along with a signature file. In the subsequent section, we will see how to verify the signature to make sure you have an authentic XRd tarball from Cisco.


Control Plane image: [https://software.cisco.com/download/home/286331236/type/280805694](https://software.cisco.com/download/home/286331236/type/280805694)  

vRouter image: [https://software.cisco.com/download/home/286331238/type/280805694/](https://software.cisco.com/download/home/286331238/type/280805694/)
{: .notice--success}

## Download
