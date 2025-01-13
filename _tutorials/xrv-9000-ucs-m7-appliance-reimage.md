---
published: true
date: '2025-01-13 14:26 +0100'
title: Cisco IOS XRv 9000 UCS M7 appliance reimage
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
# Introduction
This procedure aims to document the steps required to fully reimage a Cisco IOS XRv 9000 Appliance running IOS XR 24.4.1 and on Cisco UCS M7 series. 
Reimage can be used to fully reinstall the system from scratch for staging or system recovery reasons. 
Reimage will remove all files and configuration from the system. Perform backup accordingly before executing the procedure.

# Prerequisites
It’s important to check UCS appliance is healthy before proceeding further. This can be verified on the CIMC interface and no faults should be reported:

Review faults and logs and ensure there is anything suspicious:

# Cisco IOS XRv 9000 ISO Download
IOS XR 24.4.1 image can be downloaded from this location. The file fullk9-R-XRV9000-2441-RR.tar must be used: it contains k9 package for crypto features (e.g SSH), and the image is optimized for a BGP Route-Reflector function (72GB of RAM is dedicated to the virtual Route Processor, 16GB for the virtual Line Card, overall system leverages 28 cores from the Intel(R) Xeon(R) Gold 5420+ CPU)

