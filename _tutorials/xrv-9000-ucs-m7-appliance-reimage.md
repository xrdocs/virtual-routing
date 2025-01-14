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
# Objectives
This procedure aims to document the steps required to fully reimage a Cisco IOS XRv 9000 Appliance running IOS XR 24.4.1 and on Cisco UCS M7 series. 
Reimage can be used to fully reinstall the system from scratch for staging or system recovery reasons. 
Reimage will remove all files and configuration from the system. Perform backup accordingly before executing the procedure.

# Cisco IOS XRv 9000 UCS M7 appliance introduction

The Cisco IOS XRv 9000 UCS M7 appliance is powered by Cisco UCS C220 M7 series and Cisco IOS-XR 64bit. It is perfectly suitable for BGP Route-Reflector (RR) and Path Computation Element (PCE) use cases.

Benefits:
- Proven architecture: robust IOS-XR field-proven BGP stack
- Scale and performance
- Fully integrated solutions
- Native 100G connectivity, with ability to support 100G LR4 optics
- Power consumption: 250W

There are two references available:
- XRV-M7-APLN-25G: 4x10G/25G ports available on a single NIC (Cisco-Intel E810XXVDA4L 4x25/10 GbE SFP28 PCIe NIC)
- XRV-M7-APLN-100G: 4x100G ports available on 2 x NICs (Cisco-MLNX MCX623106AS-CDAT 2x100GbE QSFP56 PCIe NIC)

# Prerequisites
It’s important to check UCS appliance is healthy before proceeding further. This can be verified on the CIMC interface and no faults should be reported:

Review faults and logs and ensure there is anything suspicious:

# Cisco IOS XRv 9000 ISO Download
IOS XR 24.4.1 image can be downloaded from this location. The file fullk9-R-XRV9000-2441-RR.tar must be used: it contains k9 package for crypto features (e.g SSH), and the image is optimized for a BGP Route-Reflector function (72GB of RAM is dedicated to the virtual Route Processor, 16GB for the virtual Line Card, overall system leverages 28 cores from the Intel(R) Xeon(R) Gold 5420+ CPU)

# Virtual Media Mapping

# Boot Order Update

# Appliance Reboot

# Video
The whole process is illustrated in following video:

# Conclusion
This document covered reimage procedure which can be used to stage or recover an IOS XRv 9000 appliance based on Cisco UCS M7 server. While IOS XR 24.4.1 was used to illustrate it, the method will be similar for upcoming software releases.