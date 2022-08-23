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


## XRd Form Factors

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


**XRd Control Plane image on CCO:** [https://software.cisco.com/download/home/286331236/type/280805694](https://software.cisco.com/download/home/286331236/type/280805694)  
{: .notice--success}  

**XRd vRouter image on CCO:** [https://software.cisco.com/download/home/286331238/type/280805694/](https://software.cisco.com/download/home/286331238/type/280805694/)
{: .notice--success}


You will need to login with a valid cisco account that has privileges to download Cisco software images. If your Cisco account does not have the required privileges to download these images, please contact your Cisco Sales Representative.
{: .notice--danger}


### Browse to the CCO Download URL

For the Control Plane image, browse to [https://software.cisco.com/download/home/286331236/type/280805694](https://software.cisco.com/download/home/286331236/type/280805694) 

![cco_url.png]({{base_path}}/images/cco_url.png)


As of August 2022, the lstest XRd release is 7.7.1.

Click on the Download link next to the .tgz artifact:

![cco_url_download.png]({{base_path}}/images/cco_url_download.png)


At this stage you will be prompted to login with a cisco.com account with a valid service account associated:

![login_service_contract_prompt.png]({{base_path}}/images/login_service_contract_prompt.png)  

Click on Login and go through the typical Cisco SSO login flow:


![cco_sso_login.png]({{site.baseurl}}/images/cco_sso_login.png)


Assuming your cisco.com account has the required privileges, you will be prompted to Accept the License agreement associated with the image:

[accept_license_agreement.png({{base_path}}/images/accept_license_agreement.png) 





## Download
