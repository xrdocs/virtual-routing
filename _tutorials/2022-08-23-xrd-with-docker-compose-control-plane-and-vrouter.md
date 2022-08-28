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
In [Part-5]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter) of this tutorial series, we learnt how to launch XRd (both Control-Plane and vRouter formats) in a standalone manner using docker. This involved initial bring-up with required options passed to docker for successful XRd boot, access to the CLI and bash shells, bootstrap configuration and ZTP automation capabilities and establishment of external connectivity by providing macvlan interfaces (XRd control-plane) or pci network devices (XRd vRouter) to XRd instances.  


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

As mentioned above, `xr-compose` introduces its own language + syntax on top of the generic docker-compose YAML syntax. Before we attempt to use the 














Part-7 of the XRd tutorials Series here: [Setting up k8s using KIND for XRd]({{base_path}}/2022-08-23-setting-up-kubernetes-using-kind-for-xrd).
