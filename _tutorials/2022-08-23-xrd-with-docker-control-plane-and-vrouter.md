---
published: true
date: '2022-08-23 04:52 +0530'
title: 'XRd with docker: Control-Plane and vRouter'
author: Akshat Sharma
excerpt: Learn the basics of bringing up XRd with Docker.
position: hidden
tags:
  - iosxr
  - cisco
  - virtual
  - vm
  - container
  - docker
  - kubernetes
  - xrd
  - xrv9k
  - xrd-tutorial-series
---

{% include base_path %}
{% include toc %}

* This is Part-4 of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series).   
* Skip to Part-5 here: [XRd with Docker-compose: Control-Plane and vRouter]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-compose-control-plane-and-vrouter). 
* Re-read Part-3 here: [Setting up the Host Environment to run XRd]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd)



## Introduction


In [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them) of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series), we learnt how to fetch the XRd images from software.cisco.com (CCO) and verified their signatures.  
Then in [Part-3]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd), we set up the host environment required to run XRd (both variants - Control-Plane and vRouter) and also installed docker and docker-compose as part of the Host machine setup.  

## XRd-Tools launch-xrd Script

In this tutorial, we will focus on the `launch-xrd` script that we introduced as part of the xrd-tools repository in [Part-2]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd/) of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series).  

While XRd images can be launched using docker natively - `launch-xrd` acts as a great abstraction to simplify the launching process for XRd images. It is essentially a wrapper around `docker run` with all the required options to boot XRd.

**Note**: A general guidance on the use of any xrd-tools script is to utilize the `--help` option to first dump the list of options available for use with each script. In these tutorials, we will attempt to try the most important/common options but the reader is encouraged to follow the help blurbs and try each option for each of the scripts.
{: .notice--info}  



## Launch XRd using docker


Before we begin, let's load the XRd images we downloaded from CCO into the local docker daemon.

The image tarballs we expanded in [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them) are dumped below.  

```
cisco@xrdcisco:~/images$ tree .
.
├── xrd-control-plane
│   ├── IOS-XR-SW-XRd.crt
│   ├── cisco_x509_verify_release.py3
│   ├── cisco_x509_verify_release.py3.README
│   ├── cisco_x509_verify_release.py3.signature
│   ├── xrd-control-plane-container-x64.7.7.1.tgz
│   ├── xrd-control-plane-container-x64.dockerv1.tgz
│   └── xrd-control-plane-container-x64.dockerv1.tgz.signature
└── xrd-vrouter
    ├── IOS-XR-SW-XRd.crt
    ├── cisco_x509_verify_release.py3
    ├── cisco_x509_verify_release.py3.README
    ├── cisco_x509_verify_release.py3.signature
    ├── xrd-vrouter-container-x64.7.7.1.tgz
    ├── xrd-vrouter-container-x64.dockerv1.tgz
    └── xrd-vrouter-container-x64.dockerv1.tgz.signature

2 directories, 14 files
cisco@xrdcisco:~/images$ 
```



Part-5 of the XRd tutorials Series: [here]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-compose-control-plane-and-vrouter).
