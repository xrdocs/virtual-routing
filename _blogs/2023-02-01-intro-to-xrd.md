---
published: true
date: '2023-02-01 14:10 -0800'
title: Intro to XRd
tags:
  - iosxr
  - cisco
  - linux
  - xrd
  - container
  - docker
  - kubernetes
  - virtual
author: Raja Kolagatla
position: hidden
---
{% include base_path %}
{% include toc %}
# Technology state and Market trends

Network Function Virtualization (NFV) has been around for years. The
Virtualization exercise which has been started to utilize the compute
resources in an efficient way has been adopted by the Network functions
to deploy the networks in much more agile & cost-effective manner.

![]({{site.baseurl}}/images/xrd_intro_figure1.png){width="3.468801399825022in"
height="2.7555555555555555in"}But the virtualization uptick has been
slow due to:

a)  ETSI MANO standards were slow in converging

b)  throughput on COTS severs is not enough for some of the Network
    Place-in-a-Networks (PINs)

c)  fragmented deployments in various virtualization environments.

However, with changing traffic consumption patterns by end users
(detailed in later sections), Service Providers are preferring the
distributed deployments with Software Defined Routing (Figure 1).
Kubernetes (K8S) has emerged as de-facto way of deploying containerized
applications, bringing in commonality among various environments on how
the virtualized resources are presented to the applications and this is
one of the key drivers for the Virtual or Cloud Native Network Function
adoption.

A recent
[blog](https://blogs.cisco.com/sp/inflection-points-of-a-converged-metro)
by Johan Gustawsson aptly summarizes the key trends in metro
architecture: "Edge computing and the hosting of virtualized network
functions, new revenue-creating applications, and localized content
drive the intense need for efficient integration into networking". The
blog also talks about "The network edge must become a floating function
implementable at any hierarchical layer to facilitate integration with
more distributed computing and content hosting." This trend is not
common to Metro architecture but universally applies to entirety of
Service Provider Networks. As more content and applications need to be
deployed closer to the end user, Service Providers are forced to extend
the compute capabilities to at farther points in the network. VNFs play
a significant role in connecting & processing of the traffic from the
applications and content caches to the Data Centres or public clouds
where bulk of application processing is done. Figure 2 details the
relevant VNFs, that can be deployed at different Edge locations, how the
play of Service Providers and Public Cloud Providers can span across the
edge locations in the network.![Graphical user interface Description
automatically generated]({{site.baseurl}}/images/xrd_intro_figure2.png){width="6.268055555555556in"
height="2.9027777777777777in"}

Figure 2

Public Cloud Providers (PCPs) are providing solutions to install the
infrastructure at on-prem or Edge locations, in order to extend the same
tooling and management for the applications that run-in the cloud,
providing consistent hybrid cloud experience. In this process, PCPs are
partnering with telcos to innovate new ways of connectivity and offer
new Value-Added Services to end users. In this partnership, Service
Providers benefit from:

a\) PCPs' vast experience in managing virtual and cloud native workloads

b\) quicker, on-demand scale-up and scale-down options provided by PCPs.

Figure 3![]({{site.baseurl}}/images/xrd_intro_figure3.png){width="3.865972222222222in"
height="2.9965277777777777in"} indicates the speed improvements done on
Intel x86 CPUs from past few generations. Though the throughput (and
Packets Processed per Second -- PPS) augmentation does not follow
Moore's law anymore, there is a considerable jump in the throughput
numbers, bringing cost per Gbps down. In addition to above, there are
considerable improvements in I/O and L2/L3 caches in the newer
generation x86 processors. These improvements will remove the
bottlenecks and drive more Network Functions to be Virtualized.

# Containerized XR overview

XRd is the latest containerized Virtual Router offering from Cisco,
deployable on the Containerized Infra or Cloud Infra along with other
Virtual or Cloud Network Functions and applications.

XRd complements the Physical routers, in Virtual or Cloud environments,
to provide an end-to-end ubiquitous XR experience. All the
programmability aspects (NETCONF and YANG models) are inherited from
IOS-XR to XRd, which allows operators to reuse the same automation and
orchestration systems, presenting seamless visualization and management
of XR devices, physical or virtual, deployed on-prem cloud or public
cloud. XRd comes with support for low footprint optimization and can run
with as low as 2 CPUs.

XRd is being offered in two flavours:

-   Control plane (with minimal forwarding plane functionality) which is
    used for signalling heavy applications like Route reflector, SR-PCE.

-   vRouter (Control plane and the Forwarding plane bundled) which is
    suggested for Provider Edge

Apart from legacy/traditional use cases of Virtual Route Reflector
(vRR), Path Controller Element (PCE), Virtual Provider Edge (vPE) and
lab simulation, XRd brings in support for new use cases such as Virtual
Cell Site Router (vCSR) and Cloud Router Gateway.

XRd can be deployed as Virtual Cell Site Router (vCSR) with very low
footprint sharing the compute environment with Virtual DU (vDU), without
a need for dedicated Cell Site Router (Figure 3). With 2 vCPUs, vCSR can
crank up to 30Gbps (depends on actual CPU & average packet size). Since
vDU is a CPU hungry application, the low footprint of XRd helps remove
the need for installing a dedicated router at the sites which do not
need massive MIMO scale throughputs and have limited power & space. For
those sites, physical cell site router like NCS5xx is needed to satisfy
the port count needed for Front haul (detailed in Figure 4 below).

![Diagram Description automatically
generated]({{site.baseurl}}/images/xrd_intro_figure4.png){width="6.268055555555556in"
height="3.2954735345581803in"}

Figure 4

XRd as a containerized application can run on Public Cloud Infra and can
be deployed as Cloud Router Gateway (Figure 5), enabling taking the
on-prem transport protocols into the public cloud. This allows operators
to extend their existing monitoring and assurance systems to the Network
Functions, deployed in the public cloud, providing unified XR
experience. XRd provides an efficient overlay solution masking some of
the limitations of the Public Cloud Infra, which does not support
broadcast & multicast capabilities which inhibit running traditional IGP
protocols. XRd Cloud router provides the overlay solution, to uplift the
transmission of traditional SP protocol packets via the GRE tunnels to
the VPCs in the public cloud. With this overlay solution, Service
providers can seamlessly manage the telco applications and routing
instances across the existing network and public cloud, using the
current automation and assurance systems. Currently the Cloud Router
solution is being tested by one of the largest green field mobile
operator networks in US, where the 5G Packet core is deployed in AWS and
Cloud Router acts as gateway for the 5GC applications. In future, once
the support for IPv6 underlay comes from PCP, SRv6 to avoid complexity
of various layers inside GRE tunnels.

![]({{site.baseurl}}/images/xrd_intro_figure5.png){width="6.530555555555556in"
height="2.7893383639545055in"}

Figure 5

As the cloud computing capabilities extend to the Far Edge, XRd can play
an increased role either for on-ramping traffic to CDNs, Caches or to
act as a gateway for Distributed Public Cloud locations (e.g., Amazon
Outposts).

# Summary and call for action

With release 781, XRd is qualified for vanilla K8S and Amazon EKS
environments. Helm chart support has been added as part of the current
781 release, making it easily deployable in various Cloud Infra
environments. ENA NICs are supported for XRd. Earlier release of 771
(FCS for XRd), supports docker deployment for lab evaluation. Intel E810
and 700 series are available for evaluation, though official support is
pending.

Relevant cloud formation templates for the EKS deployment are available
in the public GitHub page of [XRd](https://github.com/ios-xr/xrd-tools).
Currently the team is working with other Cloud Infra providers to
qualify XRd in other host environments.

More technical details on the XRd deployment, are documented in
[tutorial](https://xrdocs.io/virtual-routing/tutorials/) sections of
XRdocs. XRd is available for 90-day trial from
[CCO](https://software.cisco.com/download/home/286331238/type/280805694/release/7.8.1).
Please go ahead and spin up few XRd instances in your lab and send
feedback to us!
