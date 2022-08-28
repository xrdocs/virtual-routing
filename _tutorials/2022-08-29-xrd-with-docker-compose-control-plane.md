---
published: true
date: '2022-08-23 05:00 +0530'
title: 'XRd with docker-compose: Control-Plane'
author: Akshat Sharma
tags:
  - iosxr
  - linux
  - xrd-tutorial-series
  - docker
  - virtual
  - container
  - vm
  - xrd
  - xrv9k
  - kubernetes
position: top
excerpt: Learn how to bring up XRd with docker-compose for the Control-Plane variant.
---
{% include base_path %}
{% include toc %}

* This is Part-6 of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series)  
* Skip to Part-7 here: [Setting up k8s using KIND for XRd]({{base_path}}/tutorials/2022-08-23-setting-up-kubernetes-using-kind-for-xrd). 
* Re-read Part-5 here: [Xrd with Docker: Control-Plane and vRouter]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter)

## Introduction


In [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them) of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series), we learnt how to fetch the XRd images from software.cisco.com (CCO) and verified their signatures.  
Then in [Part-3]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd), we set up the host environment required to run XRd (both variants - Control-Plane and vRouter) and also installed docker and docker-compose as part of the Host machine setup. In this tutorial, we will leverage the docker-compose installation and learn to bring up some sample XRd compose topologies.  
In [Part-5]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter) of this tutorial series, we learnt how to launch XRd (both Control-Plane and vRouter formats) in a standalone manner using docker. This involved learning about:
* Initial bring-up with required options passed to docker for successful XRd boot
* Access to the CLI and bash shells
* Bootstrap configuration and ZTP automation capabilities and 
* Establishment of external connectivity by providing macvlan interfaces (XRd control-plane) or pci network devices (XRd vRouter) to XRd instances.  

In this tutorial we will take this knowledge a bit further by leveraging [docker-compose](https://docs.docker.com/compose/) instead of standalone [docker](https://docs.docker.com/get-started/overview/).  While docker-compose leverages docker behind-the-scenes - it is more of a "container-system orchestrator" or a "topology orchestrator" for docker containers. In essence, it greatly simplifies the specification of individual container settings with a custom YAML syntax, abstracts network creation - bridges, macvlan interfaces etc. and the connection of containers to these networks, and helps articulate inter-container relationships in simplified manner. 


## XRd-Tools xr-compose Script


In this tutorial, we will focus on the `xr-compose` script that we introduced as part of the xrd-tools repository in [Part-2]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd/) of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series).  

The xr-compose script is a wrapper around [docker-compose](https://docs.docker.com/compose/). In addition to the general docker-compose YAML syntax (see https://docs.docker.com/compose/compose-file/), xr-compose also supports some XR-specific fields that 'expand out' to include all the fields required to hide implementation-specific details from the user. It also takes care of boilerplate docker-compose items that are desired for every XR container service.

**Note**: A general guidance on the use of any xrd-tools script is to utilize the `--help` option to first dump the list of options available for use with each script. In these tutorials, we will attempt to try the most important/common options but the reader is encouraged to follow the help blurbs and try each option for each of the scripts.
{: .notice--info}  


Dumping the available options with `xr-compose`:   

```
cisco@xrdcisco:~/xrd-tools/scripts$ ./xr-compose --help
usage: xr-compose [-h] [-f FILE] [-o FILE] [-i IMAGE] [-t STR] [-l] [-m PATH [PATH ...]] [-d] [--privileged]

Translate xr-compose input YAML into the full YAML required to run XRd topologies using docker-compose. Specify '--launch' to additionally launch the topology (otherwise
docker-compose can be run directly with the output YAML). Note that an image must be specified with -i if images are not specified in the YAML input for each service.

optional arguments:
  -h, --help            show this help message and exit
  -f FILE, --input-file FILE
                        Specify an alternative input file.
  -o FILE, --output-file FILE
                        Specify an alternative output file.
  -i IMAGE, --image IMAGE
                        Name/ID of loaded XRd image to launch. This will be overridden by any images specified in the input YAML.
  -t STR, --topo-id STR
                        Specify a topology instance identifier used to suffix container, volume, and network names.
  -l, --launch          Launch a topology from the generated docker-compose YAML.
  -m PATH [PATH ...], --mount PATH [PATH ...]
                        A space separated list of paths to mount into each XR container. Relative paths will be treated as relative to the input YAML file. Each path can be of
                        the form '<src>' or '<src>:<tgt>'.
  -d, --debug           Enable debug output
  --privileged          Launch in privileged mode
cisco@xrdcisco:~/xrd-tools/scripts$ 

```


## Understanding xr-compose YAML fields

As mentioned above, `xr-compose` introduces its own language + syntax on top of the generic docker-compose YAML syntax. Before we attempt to use the `xr-compose` script, let's take a look at 
an xr-compose yaml template file provided in the `/templates` directory of the `xrd-tools` repository that we cloned to our host machine in [Part-2]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd/) of the XRd tutorials Series.  

Dumping the contents of `docker-compose.template.xr.yml`:  


```bash
cisco@xrdcisco:~/xrd-tools/templates$ cat docker-compose.template.xr.yml 
# Copyright 2020-2022 Cisco Systems Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.


# Introduction
# ------------
# This file documents the schema for the input YAML file to be passed to the
# xr-compose script.
#
# xr-compose translates the XR keywords documented in this template to the full
# docker-compose YAML that is used to bring up topologies of XRd Docker
# containers.
#
# This file is not intended as a bootable sample topology, e.g. it does not
# refer to real config file paths.
#
# See samples/ for some fully-formed examples that can be booted.
#

# Notes
# -----
#  - Any docker-compose fields are valid e.g. 'image' and 'container_name'
#    fields may be specified for an XR service.
#
#  - See the docker-compose reference
#    (https://docs.docker.com/compose/compose-file/) for more information on
#    the docker-compose YAML schema.
#
#  - The added XR keywords all start with 'xr_', with the one exception of the
#    'non_xr' field.
#
#  - XR keywords will be expanded out to valid docker-compose YAML by
#    xr-compose.
#
#  - Boilerplate docker-compose fields such as 'version', 'stdin_open', 'tty',
#    and 'privileged' that are required for each container will be filled in
#    with default values if not specified.
#
#  - Each container will have a volume generated to store data that should be
#    persistent across multiple boots.
#

services:
  xr-1:
    # The loaded XRd image must be specified either here or using the '--image'
    # CLI option to xr-compose.
    image: ios-xr:7.4.1
    # A container name will be generated from the service name and the topology
    # identifier if one is not specified.
    # The topology identifier may be specified as an input argument to
    # xr-compose, otherwise being generated with the format <username>-<cwd>.
    container_name: xr-1
    # Optionally specify a path to startup XR config for this service. Relative
    # paths are interpreted as relative to the input YAML file.
    xr_startup_cfg: /path/to/config_file_xr1
    # Optionally specify a path to a boot script to be run after all startup
    # configuration has been applied
    xr_boot_script: /path/to/boot_script
    # Optionally specify XR interfaces for this service to have. Valid
    # interfaces are currently:
    # - GigabitEthernet interfaces:
    #     Gi0/0/0/x
    # - MgmtEthernet interfaces:
    #     Mg0/RP0/CPU0/0
    # The following optional per-interface flags may be set:
    # - chksum: This interface should have checksum offload counteract enabled.
    #     This defaults to True for any interfaces in non-L2 networks
    #     (predefined Docker network), in anticipation of XR to non-XR
    #     connectivity which requires the counteract behavior.
    # - snoop_v[4|6]: This interface should have XR interface IPv4/v6 address
    #     configuration added to it, using the IP addresses assigned by the
    #     container orchestrator. Defaults to False.
    # - snoop_v[4|6]_default_route: This interface should have XR IPv4/6
    #     default route configuration added, using the default route assigned
    #     by the container orchestrator. Defaults to False.
    xr_interfaces:
      - Gi0/0/0/0:
          snoop_v4: True
          snoop_v6: True
      - Gi0/0/0/1:
          chksum: False
      - Gi0/0/0/2:
          chksum: False
      - Mg0/RP0/CPU0/0:
          snoop_v4: True
          snoop_v6: True
          snoop_v4_default_route: True
          snoop_v6_default_route: True
    # Specified IP addresses for XR interfaces 'reserve' this address within
    # the Docker network. The same address will need to be configured in XR
    # on the interface.
    networks:
      mgmt:
        ipv4_address: 17.19.0.2
  xr-2:
    image: ios-xr:7.4.1
    xr_startup_cfg: /path/to/config_file_xr2
    xr_interfaces:
      - Gi0/0/0/0
      - Gi0/0/0/1
      - Gi0/0/0/2
      - Mg0/RP0/CPU0/0
  xr-3:
    image: ios-xr:7.4.1
    xr_startup_cfg: /path/to/config_file_xr3
    xr_interfaces:
      - Gi0/0/0/0
      - Gi0/0/0/1
  xr-4:
    image: ios-xr:7.4.1
    xr_startup_cfg: /path/to/config_file_xr4
    xr_interfaces:
      - Gi0/0/0/0
  ubuntu-1:
    # Services annotated with the 'non_xr' keyword will be left unchanged by
    # xr-compose.
    non_xr: true
    image: ubuntu:20.04
    container_name: ubuntu-1
    tty: true
    stdin_open: true
    cap_add:
       - NET_ADMIN
    networks:
      xrd-1-ubuntu-1:
        ipv4_address: 10.0.0.2

# Specify L2 connections for XR interfaces, to be set up using Docker networks.
# Each interface may be included in at most one network, and each network
# may include at most one interface from any given XR service.
# Interfaces not added to any network will have their own Docker network
# created to supply an interface, but will not be connected to any
# other containers.
# Note that the syntax here corresponds to a list of lists - a list of networks,
# each represented as a list of interfaces belonging to specified containers.
xr_l2networks:
  - ["xr-1:Gi0/0/0/0", "xr-2:Gi0/0/0/0"]
  - ["xr-1:Gi0/0/0/1", "xr-3:Gi0/0/0/0"]
  - ["xr-2:Gi0/0/0/1", "xr-3:Gi0/0/0/1", "xr-4:Gi0/0/0/0"]

networks:
  mgmt:
    ipam:
      config:
        - subnet: 172.19.0.0/24
    # Interfaces may be added to predefined Docker networks, if they
    # are not included in any xr_l2network. This may be desirable
    # for linking non-XR containers to XR containers, and for ensuring
    # the network subnet matches the interface IP address, so that management
    # interfaces are accessible.
    xr_interfaces:
      - xr-1:Mg0/RP0/CPU0/0
      - xr-2:Mg0/RP0/CPU0/0
  xrd-1-ubuntu-1:
    ipam:
      config:
        - subnet 10.0.0.0/16
    xr_interfaces:
      - xr-1:Gi0/0/0/2
cisco@xrdcisco:~/xrd-tools/templates$ 


```




### XR-Compose YAML - Services
The following XR-specific items are accepted in the 'service' section:

#### xr_startup_cfg
* Startup config filename.
* Non-qualified filenames are assumed relative to the directory containing the topology YAML.
* If not specified, no startup config is mounted.

```
Syntax example for xr_startup_cfg
 services:
   xr-1:
     xr_startup_cfg: xrd-1-startup.cfg
```  


#### xr_interfaces
* List of interfaces that the container will have in the form Gi0/0/0/x, or Mg0/RP0/CPU0/0.
* Gi interfaces must be added in a continuous sequence, e.g. cannot specify 0, 2, 3.
* If not specified, the XR container will not have any interfaces.
* Each interface must be listed either under networks or xr_l2networks - not both.
* If an interface is not listed under in either networks or xr_l2networks, then it will be created without linking to anything.  

```
Syntax example for xr_interfaces at container-level
 services:
   xr-1:
     xr_interfaces:
       - Gi0/0/0/0
       - Gi0/0/0/1
       - Gi0/0/0/2
       - Mg0/RP0/CPU0/0
```  


### XR-Compose YAML - Networks  

The following XR-specific items may be added to a 'network' section:

#### xr_interfaces
* List of XR interfaces.
* Used if an XR interface must be accessible from the host (e.g. for management) or accessible from a non-XRd container.
* Interfaces must be specified in the following format: service-name:interface-name, e.g. xr-1:GE0/0/0/0.  

```
Syntax example for xr_interfaces at network-level
networks:
   xrd-1-ubuntu-1:
     xr_interfaces:
       - "xr-1:Gi0/0/0/2"
```   


### XR-Compose YAML - Top-level

The following top-level XR-specific items may be specified:

#### xr_l2networks
* List of interface lists.
* Used to specify which XR container interfaces should be joined together on the same L2 network.
* Interfaces must be specified in the following format: service-name:interface-name, e.g. xr-1:GE0/0/0/0.
* One L2 network must not include more than one interface from any one container.

```
Syntax example for xr_l2networks at top-level
 xr_l2networks:
   - ["xr-1:Gi0/0/0/0", "xr-2:Gi0/0/0/0"]
   - ["xr-1:Gi0/0/0/1", "xr-3:Gi0/0/0/0"]
   - ["xr-2:Gi0/0/0/1", "xr-3:Gi0/0/0/1", "xr-4:Gi0/0/0/0"]
```  





## Launching Sample XRd docker-compose Topologies


In the `xrd-tools/` repository we cloned to the host machine in [Part-2]({{base_path}}/tutorials/2022-08-22-helper-tools-to-get-started-with-xrd/), there is a `samples/` directory that contains some docker-compose.xr.yaml examples. Let's work through an example to better understand how to use `xr-compose`:  

```
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos$ pwd
/home/cisco/xrd-tools/samples/xr_compose_topos
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos$  tree .
.
├── bgp-ospf-triangle
│   ├── docker-compose.xr.yml
│   ├── topo-diagram.png
│   ├── xrd1_xrconf.cfg
│   ├── xrd2_xrconf.cfg
│   └── xrd3_xrconf.cfg
├── segment-routing
│   ├── docker-compose.xr.yml
│   ├── xrd-1-startup.cfg
│   ├── xrd-2-startup.cfg
│   ├── xrd-3-startup.cfg
│   ├── xrd-4-startup.cfg
│   ├── xrd-5-startup.cfg
│   ├── xrd-6-startup.cfg
│   ├── xrd-7-startup.cfg
│   └── xrd-8-startup.cfg
└── simple-bgp
    ├── docker-compose.xr.yml
    ├── xrd-1_xrconf.cfg
    └── xrd-2_xrconf.cfg

3 directories, 17 files
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos$ 
```



Let's take a look `simple-bgp`:  


```bash

cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos$ cd simple-bgp/
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ ls
docker-compose.xr.yml  xrd-1_xrconf.cfg  xrd-2_xrconf.cfg
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ cat docker-compose.xr.yml 
# Copyright 2020-2022 Cisco Systems Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Topology
# source <--> xr-1 <--> xr-2 <--> dest

# IP addresses
# source:            10.1.1.2
# xr-1-GE0 (left ):  10.1.1.3
# xr-1-GE1 (right:   10.2.1.2
# xr-2-GE0 (left):   10.2.1.3
# xr-2-GE1 (right):  10.3.1.2
# dest:              10.3.1.3

services:
  source:
    non_xr: true
    image: alpine:3.15
    container_name: source
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
    command: /bin/sh -c "ip route add 10.0.0.0/8 via 10.1.1.3 && /bin/sh"
    networks:
      source-xrd-1:
        ipv4_address: 10.1.1.2
  xr-1:
    xr_startup_cfg: xrd-1_xrconf.cfg
    xr_interfaces:
      - Gi0/0/0/0
      - Gi0/0/0/1
      - Mg0/RP0/CPU0/0
    networks:
      source-xrd-1:
        ipv4_address: 10.1.1.3
  xr-2:
    xr_startup_cfg: xrd-2_xrconf.cfg
    xr_interfaces:
      - Gi0/0/0/0
      - Gi0/0/0/1
      - Mg0/RP0/CPU0/0
    networks:
      xrd-2-dest:
        ipv4_address: 10.3.1.2
  dest:
    non_xr: true
    image: alpine:3.15
    container_name: dest
    stdin_open: true
    tty: true
    networks:
      xrd-2-dest:
        ipv4_address: 10.3.1.3
    cap_add:
      - NET_ADMIN
    command: /bin/sh -c "ip route add 10.0.0.0/8 via 10.3.1.2 && /bin/sh"

xr_l2networks:
  - ["xr-1:Gi0/0/0/1", "xr-2:Gi0/0/0/0"]
networks:
  mgmt:
    xr_interfaces:
      - xr-1:Mg0/RP0/CPU0/0
      - xr-2:Mg0/RP0/CPU0/0
    ipam:
      config:
        - subnet: 172.30.0.0/24
  source-xrd-1:
    ipam:
      config:
        - subnet: 10.1.1.0/24
    xr_interfaces:
      - xr-1:Gi0/0/0/0
  xrd-2-dest:
    ipam:
      config:
        - subnet: 10.3.1.0/24
    xr_interfaces:
      - xr-2:Gi0/0/0/1
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 

```

This docker-compose topology creates the following:  

* Services consisting of 4 docker instances:
    1. `source`: An alpine Linux based docker instance that is one end of the network (source of traffic).
    2. `dest`: An alpine Linux based docker instance at the other end of the topology (destination of traffic from source)
    3. `xr-1`: First XRd Control-Plane instance of the router network in the middle, connected directly to `source`
    4. `xr-2`: Second XRd Control-Plane instance of the router network in the middle, connected directly to `dest`

* 4 Networks :

  1. `mgmt`:  The management network that each XRd instanc Mgmt port is connected to.
  2. `source-xrd-1`: Network connecting one interface of `source` and one interface of `xr-1`
  3. `xr_l2networks`: A set of l2 network connections between the routers `xr-1` and `xr-2`
  4. `xrd-2-dest`:  Network connecting one interface of `xr-2` and one interface of `dest`.
  
  
  
Further, in the same directory there are two more files - the XR configurations of xr-1 and xr-2. Dumping the config for xr-1:  


```bash
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ cat xrd-1_xrconf.cfg 
!! Copyright 2020-2022 Cisco Systems Inc.
!!
!! Licensed under the Apache License, Version 2.0 (the "License");
!! you may not use this file except in compliance with the License.
!! You may obtain a copy of the License at
!!
!! http://www.apache.org/licenses/LICENSE-2.0
!!
!! Unless required by applicable law or agreed to in writing, software
!! distributed under the License is distributed on an "AS IS" BASIS,
!! WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
!! See the License for the specific language governing permissions and
!! limitations under the License.

! Configure mgmt port
interface MgmtEth0/RP0/CPU0/0
 ipv4 address 172.30.0.2 255.255.255.0
!

! Configure left data port
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.3 255.255.255.0
!

! Configure right data port
interface GigabitEthernet0/0/0/1
 ipv4 address 10.2.1.2 255.255.255.0
!

! Configure BGP
router bgp 100
 bgp router-id 10.2.1.2
 bgp update-delay 0
 address-family ipv4 unicast
  redistribute connected
 !
 neighbor 10.2.1.3
  remote-as 100
  address-family ipv4 unicast
  !
 !
!
end


```  

The overall topology represented in the figure below:  


![simple-bgp-xr-compose.png]({{site.baseurl}}/images/simple-bgp-xr-compose.png)


If everything comes up correctly, the xr-1 and xr-2 will learn about each other's directly connected networks (source and destination) and enable traffic flow end-to-end. Pretty straightforward.


The steps to activate this environment are as follows:  

### Transform docker-compose.xr.yml to docker-compose.yml

We use the `xr-compose` script for this purpose. To create the docker-compose.yml file, run the following command:


```
xr-compose -f docker-compose.xr.yml -i <XRd control-plane image name>

```
where:

* `-f` or `--input-file`: Options to specify the input docker-compose.xr.yml file to be transformed
* '-i' or '--image': Option to specify the XRd control-plane image to use for XRd service in the compose file.



Running the command from within the `samples/` directory:

```
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ ../../../scripts/xr-compose -f docker-compose.xr.yml -i localhost/xrd-control-plane
INFO - Writing output docker-compose YAML to docker-compose.yml
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 

```


Dumping the generated `docker-compose.yml` file:  


```
networks:
  mgmt:
    driver_opts:
      com.docker.network.container_iface_prefix: xr-1
    ipam:
      config:
      - subnet: 172.30.0.0/24
  source-xrd-1:
    driver_opts:
      com.docker.network.container_iface_prefix: xr-2
    ipam:
      config:
      - subnet: 10.1.1.0/24
  xr-1-gi1-xr-2-gi0:
    driver_opts:
      com.docker.network.container_iface_prefix: xr-0
    internal: true
    name: xr-1-gi1-xr-2-gi0
  xrd-2-dest:
    driver_opts:
      com.docker.network.container_iface_prefix: xr-3
    ipam:
      config:
      - subnet: 10.3.1.0/24
services:
  dest:
    cap_add:
    - NET_ADMIN
    command: /bin/sh -c "ip route add 10.0.0.0/8 via 10.3.1.2 && /bin/sh"
    container_name: dest
    image: alpine:3.15
    networks:
      xrd-2-dest:
        ipv4_address: 10.3.1.3
    stdin_open: true
    tty: true
  source:
    cap_add:
    - NET_ADMIN
    command: /bin/sh -c "ip route add 10.0.0.0/8 via 10.1.1.3 && /bin/sh"
    container_name: source
    image: alpine:3.15
    networks:
      source-xrd-1:
        ipv4_address: 10.1.1.2
    stdin_open: true
    tty: true
  xr-1:
    cap_add:
    - CHOWN
    - DAC_OVERRIDE
    - FSETID
    - FOWNER
    - MKNOD
    - NET_RAW
    - SETGID
    - SETUID
    - SETFCAP
    - SETPCAP
    - NET_BIND_SERVICE
    - SYS_CHROOT
    - KILL
    - AUDIT_WRITE
    - SYS_NICE
    - SYS_ADMIN
    - SYS_RESOURCE
    - NET_ADMIN
    - SYS_PTRACE
    - IPC_LOCK
    cap_drop:
    - all
    container_name: xr-1
    devices:
    - /dev/fuse
    - /dev/net/tun
    environment:
      XR_EVERY_BOOT_CONFIG: /etc/xrd/startup.cfg
      XR_INTERFACES: linux:xr-20,xr_name=Gi0/0/0/0,chksum;linux:xr-00,xr_name=Gi0/0/0/1
      XR_MGMT_INTERFACES: linux:xr-10,xr_name=Mg0/RP0/CPU0/0,chksum
    image: localhost/xrd-control-plane
    networks:
      mgmt: null
      source-xrd-1:
        ipv4_address: 10.1.1.3
      xr-1-gi1-xr-2-gi0: null
    security_opt:
    - apparmor:unconfined
    - label:disable
    stdin_open: true
    tty: true
    volumes:
    - source: ./xrd-1_xrconf.cfg
      target: /etc/xrd/startup.cfg
      type: bind
    - xr-1:/xr-storage/
    - read_only: true
      source: /sys/fs/cgroup
      target: /sys/fs/cgroup
      type: bind
  xr-2:
    cap_add:
    - CHOWN
    - DAC_OVERRIDE
    - FSETID
    - FOWNER
    - MKNOD
    - NET_RAW
    - SETGID
    - SETUID
    - SETFCAP
    - SETPCAP
    - NET_BIND_SERVICE
    - SYS_CHROOT
    - KILL
    - AUDIT_WRITE
    - SYS_NICE
    - SYS_ADMIN
    - SYS_RESOURCE
    - NET_ADMIN
    - SYS_PTRACE
    - IPC_LOCK
    cap_drop:
    - all
    container_name: xr-2
    devices:
    - /dev/fuse
    - /dev/net/tun
    environment:
      XR_EVERY_BOOT_CONFIG: /etc/xrd/startup.cfg
      XR_INTERFACES: linux:xr-00,xr_name=Gi0/0/0/0;linux:xr-30,xr_name=Gi0/0/0/1,chksum
      XR_MGMT_INTERFACES: linux:xr-10,xr_name=Mg0/RP0/CPU0/0,chksum
    image: localhost/xrd-control-plane
    networks:
      mgmt: null
      xr-1-gi1-xr-2-gi0: null
      xrd-2-dest:
        ipv4_address: 10.3.1.2
    security_opt:
    - apparmor:unconfined
    - label:disable
    stdin_open: true
    tty: true
    volumes:
    - source: ./xrd-2_xrconf.cfg
      target: /etc/xrd/startup.cfg
      type: bind
    - xr-2:/xr-storage/
    - read_only: true
      source: /sys/fs/cgroup
      target: /sys/fs/cgroup
      type: bind
version: '2.4'
volumes:
  xr-1:
    name: xr-1
  xr-2:
    name: xr-2
```


This `docker-compose.yml` can be used to spin up the topology using standard `docker-compose` that we installed on the host machine earlier.


### Launch the docker-compose topology


To launch, simply run `docker-compose up -d` in the same directory where the `docker-compose.yml` file was created:


```bash
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ docker-compose  up -d
Creating network "simple-bgp_xrd-2-dest" with the default driver
Creating network "simple-bgp_source-xrd-1" with the default driver
Creating network "simple-bgp_mgmt" with the default driver
Creating network "xr-1-gi1-xr-2-gi0" with the default driver
Creating xr-1   ... done
Creating dest   ... done
Creating source ... done
Creating xr-2   ... done
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 

```


With the topology successfully up, let's look at the running containers:

```bash

cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ docker ps
CONTAINER ID   IMAGE                         COMMAND                  CREATED          STATUS          PORTS     NAMES
78a60ba46388   localhost/xrd-control-plane   "/bin/sh -c /sbin/xr…"   13 seconds ago   Up 11 seconds             xr-1
c4f653b7e12f   alpine:3.15                   "/bin/sh -c 'ip rout…"   13 seconds ago   Up 11 seconds             source
5ba8faf8da35   alpine:3.15                   "/bin/sh -c 'ip rout…"   13 seconds ago   Up 12 seconds             dest
dd04360f3bc4   localhost/xrd-control-plane   "/bin/sh -c /sbin/xr…"   13 seconds ago   Up 11 seconds             xr-2
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 


```


And the created docker networks (simple-bgp*):

```
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ docker network list
NETWORK ID     NAME                      DRIVER    SCOPE
e1cca51815a4   bridge                    bridge    local
9e264ebfa892   docker-registry_default   bridge    local
14ef35eea661   host                      host      local
f42c705a30f9   none                      null      local
f149ffc2c678   simple-bgp_mgmt           bridge    local
9b7d6193098c   simple-bgp_source-xrd-1   bridge    local
648d0e57729a   simple-bgp_xrd-2-dest     bridge    local
ff98e1568ab6   xr-1-gi1-xr-2-gi0         bridge    local
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 


```


Finally, to test that things came up fine, let's exec to `xr-1` and use the ZTP automation commands to run some basic checks to see BGP neighbors and show route to confirm everything is up and running as expected:  



```
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ docker exec -it xr-1 bash
bash-4.3# 
bash-4.3# source /pkg/bin/ztp_helper.sh
bash-4.3# xrcmd "show bgp summary"
BGP router identifier 10.2.1.2, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 10
BGP main routing table version 10
BGP NSR Initial initsync version 4 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker              10         10         10         10          10           0

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
10.2.1.3          0   100       5       5       10    0    0 00:02:01          3

bash-4.3# 
bash-4.3# 
bash-4.3# xrcmd "show route 10.3.1.0/24"

Routing entry for 10.3.1.0/24
  Known via "bgp 100", distance 200, metric 0, type internal
  Installed Aug 28 20:20:40.835 for 00:02:31
  Routing Descriptor Blocks
    10.2.1.3, from 10.2.1.3
      Route metric is 0
  No advertising protos. 
bash-4.3# 
bash-4.3# 
bash-4.3# exit
exit
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ 
cisco@xrdcisco:~/xrd-to


```



```
cisco@xrdcisco:~/xrd-tools/samples/xr_compose_topos/simple-bgp$ docker exec -it source ping 10.3.1.3
PING 10.3.1.3 (10.3.1.3): 56 data bytes
64 bytes from 10.3.1.3: seq=0 ttl=62 time=8.522 ms
64 bytes from 10.3.1.3: seq=1 ttl=62 time=3.938 ms
64 bytes from 10.3.1.3: seq=2 ttl=62 time=4.199 ms
64 bytes from 10.3.1.3: seq=3 ttl=62 time=4.383 ms
^C

```


























Part-7 of the XRd tutorials Series here: [Setting up k8s using KIND for XRd]({{base_path}}/2022-08-23-setting-up-kubernetes-using-kind-for-xrd).
