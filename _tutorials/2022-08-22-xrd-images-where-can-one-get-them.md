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
  - xrd-tutorial-series
excerpt: Learn how to download the latest XRd images and set them up for further use.
---

{% include base_path %}
{% include toc %}

This is Part-1 of the [XRd tutorials Series](). Skip to Part-2 [here]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd). 

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


## Sample Download Process for the Control-Plane XRd image

For the Control Plane image, browse to [https://software.cisco.com/download/home/286331236/type/280805694](https://software.cisco.com/download/home/286331236/type/280805694) 

![cco_url.png]({{base_path}}/images/cco_url.png){: .align-center}{: .notice--primary}



As of August 2022, the lstest XRd release is 7.7.1.

Click on the Download link next to the .tgz artifact:

![cco_url_download.png]({{base_path}}/images/cco_url_download.png){: .align-center}{: .notice--primary}


At this stage you will be prompted to login with a cisco.com account with a valid service account associated:

![login_service_contract_prompt.png]({{base_path}}/images/login_service_contract_prompt.png){: .align-center}{: .notice--primary}

Click on Login and go through the typical Cisco SSO login flow:


![cco_sso_login.png]({{site.baseurl}}/images/cco_sso_login.png){: .align-center}{: .notice--primary}


Assuming your cisco.com account has the required privileges, you will be prompted to Accept the License agreement associated with the image:

![accept_license_agreement.png]({{base_path}}/images/accept_license_agreement.png){: .align-center}{: .notice--primary}

Once you accept the agreement the download should begin.



## Verify contents of the downloaded tar balls

You can check the contents of the downloaded tarballs without opening them using the `tar -t`:

<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">
cisco@xrdcisco:~/images$ tree .
.
├── xrd-control-plane
│   └── xrd-control-plane-container-x64.7.7.1.tgz
└── xrd-vrouter
    └── xrd-vrouter-container-x64.7.7.1.tgz

2 directories, 2 files
cisco@xrdcisco:~/images$ <mark>cd xrd-control-plane/</mark>
cisco@xrdcisco:~/images/xrd-control-plane$<mark> tar -tzf xrd-control-plane-container-x64.7.7.1.tgz 
cisco_x509_verify_release.py3
cisco_x509_verify_release.py3.README
cisco_x509_verify_release.py3.signature
IOS-XR-SW-XRd.crt
xrd-control-plane-container-x64.dockerv1.tgz
xrd-control-plane-container-x64.dockerv1.tgz.signature</mark>
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ <mark>cd ../xrd-vrouter/</mark>
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$<mark> tar -tzf xrd-vrouter-container-x64.7.7.1.tgz 
cisco_x509_verify_release.py3
cisco_x509_verify_release.py3.README
cisco_x509_verify_release.py3.signature
IOS-XR-SW-XRd.crt
xrd-vrouter-container-x64.dockerv1.tgz
xrd-vrouter-container-x64.dockerv1.tgz.signature</mark>
cisco@xrdcisco:~/images/xrd-vrouter$ 

</code>
</pre>
</div>


As shown above,each downloaded tarball contains the docker image tarball along with a `.signature` file that carries the cisco signature for the docker image.


## Verify Signatures


When using XRd, it is imperative to ensure you have an authentic copy of the image. Since the image is delivered as a tarball, the signature is carries in a separate `.signature` file.

To understand how to verify the signature, first let's untar the downloaded tar ball:


```bash
cisco@xrdcisco:~/images/xrd-control-plane$ tar -xzf xrd-control-plane-container-x64.7.7.1.tgz 
cisco@xrdcisco:~/images/xrd-control-plane$  tree .
.
├── IOS-XR-SW-XRd.crt
├── cisco_x509_verify_release.py3
├── cisco_x509_verify_release.py3.README
├── cisco_x509_verify_release.py3.signature
├── xrd-control-plane-container-x64.7.7.1.tgz
├── xrd-control-plane-container-x64.dockerv1.tgz
└── xrd-control-plane-container-x64.dockerv1.tgz.signature

0 directories, 7 files
cisco@xrdcisco:~/images/xrd-control-plane$ 


```

Open up the file `cisco_x509_verify_release.py3.README` which carries instructions to verify the signature for the docker tarball:


```bash
cisco@xrdcisco:~/images/xrd-control-plane$ cat cisco_x509_verify_release.py3.README
#------------------------------------------------------------------------------
# cisco_x509_verify_release.py3.README
#
# Copyright (c) 2021 by Cisco Systems, Inc.
# All rights reserved.
#------------------------------------------------------------------------------

Relevant Content
================
The following content is used for signature verification:
 1. IOS XRd container image archive
    - Cisco provided image for which signature is to be verified.
    - e.g. xrd-x64-7.5.1.dockerv1.tgz
    - In this README this will be referenced as $IMAGE_NAME

 2. Corresponding IOS XRd container image archive signature
    - Signature generated for the image
    - e.g. xrd-x64-7.5.1.dockerv1.tgz.signature
    - In this README this will be referenced as $IMAGE_SIGNATURE

 3. IOS-XR-SW-XRd End-entity certificate
    - Cisco signed x.509 end-entity certificate containing public key that can be used to
      verify the signature.
    - e.g. IOS-XR-SW-XRd.crt
    - This certificate is chained to Cisco rootCA and SubCA posted on
      - SubCA: http://www.cisco.com/security/pki/certs/xrcrrsca.cer
      - rootCA: http://www.cisco.com/security/pki/certs/crrca.cer
    - In this README this will be referenced as $EE_CERTIFICATE

 4. Cisco x509 verification script
    - Signature verification program. After downloading image,
      its digital signature, and the x.509 certificate, this program can be
      used to verify the 3-tier x.509 certificate chain and signature. Certificate
      chain validation is done by verifying the authenticity of end-entity
      certificate, using Cisco-sourced SubCA and root CA (which the script
      either reads locally or downloads from Cisco). Then this authenticated
      end-entity certificate is used to verify the signature.
    - e.g. cisco_x509_verify_release.py3
    - In this README this will be referenced as $VERIFICATION_SCRIPT

 5. README
    - This file.

=============
Requirements:
=============
1. Python 3.4.0 or later
2. OpenSSL

=========================================
How to run signature verification program:
=========================================
+Example 1 command (Cisco rootCA & subCA not local)
+--------------------------------------------------
python "$VERIFICATION_SCRIPT" -e "$EE_CERTIFICATE" -i "$IMAGE_NAME" -s "$IMAGE_SIGNATURE" -v smime --container xr --sig_type DER

Example 1 output:
-----------------
Retrieving CA certificate from http://www.cisco.com/security/pki/certs/crrca.cer ...
Successfully retrieved and verified crrca.cer.
Retrieving SubCA certificate from http://www.cisco.com/security/pki/certs/xrcrrsca.cer ...
Successfully retrieved and verified xrcrrsca.cer.
Successfully verified root, subca and end-entity certificate chain.
Successfully verified the signature of xrd-x64-7.5.1.dockerv1.tgz using IOS-XR-SW-XRd.crt

+Example 2 command (local Cisco rootCA & subCA)
+----------------------------------------------
python "$VERIFICATION_SCRIPT" -e "$EE_CERTIFICATE" -i "$IMAGE_NAME" -s "$IMAGE_SIGNATURE" -v smime --container xr --sig_type DER -c /opt/dg/pki

Example 2 output:
-----------------
Retrieving local CA certificate
Successfully retrieved and verified crrca.cer.
Retrieving local SubCA certificate
Successfully retrieved and verified xrcrrsca.cer.
Successfully verified root, subca and end-entity certificate chain.
Successfully verified the signature of xrd-x64-7.5.1.dockerv1.tgz using IOS-XR-SW-XRd.crt
cisco@xrdcisco:~/images/xrd-control-plane$ 


```


Following the above instructions, let's verify the signature on the XRd control-plane docker tarball:


```bash
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ python3 --version
Python 3.8.10
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ export VERIFICATION_SCRIPT="cisco_x509_verify_release.py3"
cisco@xrdcisco:~/images/xrd-control-plane$ export EE_CERTIFICATE="IOS-XR-SW-XRd.crt"
cisco@xrdcisco:~/images/xrd-control-plane$ export IMAGE_SIGNATURE=xrd-control-plane-container-x64.dockerv1.tgz.signature
cisco@xrdcisco:~/images/xrd-control-plane$ export IMAGE_NAME=xrd-control-plane-container-x64.dockerv1.tgz
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ python3 "$VERIFICATION_SCRIPT" -e "$EE_CERTIFICATE" -i "$IMAGE_NAME" -s "$IMAGE_SIGNATURE" -v smime --container xr --sig_type DER
Retrieving CA certificate from http://www.cisco.com/security/pki/certs/crrca.cer ...
Successfully retrieved and verified crrca.cer.
Retrieving SubCA certificate from http://www.cisco.com/security/pki/certs/xrcrrsca.cer ...
Successfully retrieved and verified xrcrrsca.cer.
Successfully verified root, subca and end-entity certificate chain.
Successfully verified the signature of xrd-control-plane-container-x64.dockerv1.tgz using IOS-XR-SW-XRd.crt
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ 

```

and similarly for the vRouter docker image tarball:


```bash
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$ export VERIFICATION_SCRIPT="cisco_x509_verify_release.py3"
cisco@xrdcisco:~/images/xrd-vrouter$ export EE_CERTIFICATE="IOS-XR-SW-XRd.crt"
cisco@xrdcisco:~/images/xrd-vrouter$ export IMAGE_SIGNATURE=xrd-vrouter-container-x64.dockerv1.tgz.signature
cisco@xrdcisco:~/images/xrd-vrouter$ export IMAGE_NAME=xrd-vrouter-container-x64.dockerv1.tgz
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$ python3 "$VERIFICATION_SCRIPT" -e "$EE_CERTIFICATE" -i "$IMAGE_NAME" -s "$IMAGE_SIGNATURE" -v smime --container xr --sig_type DER
Retrieving CA certificate from http://www.cisco.com/security/pki/certs/crrca.cer ...
Successfully retrieved and verified crrca.cer.
Retrieving SubCA certificate from http://www.cisco.com/security/pki/certs/xrcrrsca.cer ...
Successfully retrieved and verified xrcrrsca.cer.
Successfully verified root, subca and end-entity certificate chain.
Successfully verified the signature of xrd-vrouter-container-x64.dockerv1.tgz using IOS-XR-SW-XRd.crt
cisco@xrdcisco:~/images/xrd-vrouter$ 


```


There you have it- we've successfully downloaded XRd images from CCO and verified their authenticity using the packaged signature files.
{: .notice--success}

Part-2 of the XRd tutorials Series: [here]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd). 