---
published: true
date: '2022-08-23 05:00 +0530'
title: 'XRd with docker-compose: Control-Plane and vRouter'
author: Akshat Sharma
excerpt: >-
  Learn how to bring up XRd with docker-compose for the Control-Plane and the
  vRouter variants.
tags:
  - iosxr
  - cisco
  - xrd
  - xrv9k
  - virtual
  - container
  - vm
  - kubernetes
  - docker
  - xrd-tutorial-series
position: hidden
---
{% include base_path %}
{% include toc %}

* This is Part-6 of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series)  
* Skip to Part-7 here: [Setting up k8s using KIND for XRd]({{base_path}}/tutorials/2022-08-23-setting-up-kubernetes-using-kind-for-xrd). 
* Re-read Part-5 here: [Xrd with Docker: Control-Plane and vRouter]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter)

## Introduction


In [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them) of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series), we learnt how to fetch the XRd images from software.cisco.com (CCO) and verified their signatures.  
Then in [Part-3]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd), we set up the host environment required to run XRd (both variants - Control-Plane and vRouter) and also installed docker and docker-compose as part of the Host machine setup. In this tutorial, we will leverage the docker-compose installation and learn to bring up some sample XRd compose topologies.

## XRd-Tools xr-compose Script


In this tutorial, we will focus on the `xr-compose` script that we introduced as part of the xrd-tools repository in [Part-2]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd/) of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series).  












Part-7 of the XRd tutorials Series here: [Setting up k8s using KIND for XRd]({{base_path}}/2022-08-23-setting-up-kubernetes-using-kind-for-xrd).
