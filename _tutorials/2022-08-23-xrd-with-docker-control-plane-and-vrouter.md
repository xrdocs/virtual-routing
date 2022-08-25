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

Dumping the available options with `launch-xrd`:  

```bash
cisco@xrdcisco:~/xrd-tools/scripts$ ./launch-xrd --help
Usage: launch-xrd [-h|--help] [-n|--dry-run] IMG [<opts>]


Launch a single XRd container.

Use '--dry-run' to see the command that would be run without running it.

Required arguments:
  IMG                           Specify loaded container image to boot

Optional arguments:
  -f, --first-boot-config FILE  Path to startup config file for first boot
  -e, --every-boot-config FILE  Path to startup config file for every boot
  -v, --xrd-volume VOL          Name of volume used for persistency (created if
                                doesn't already exist)
  -p, --platform PLATFORM       XR platform to launch (defaults to checking the
                                image label)
  -k, --keep                    Keep the container around after it has stopped
  -c, --ctr-client EXE          Container client executable (defaults to
                                'docker'), the name is used to determine
                                whether using docker or podman
  --name NAME                   Specify container name
  --privileged                  Run the container with extended privileges
  --interfaces IF_TYPE:IF_NAME[,IF_FLAG[,...]][;IF_TYPE:IF_NAME[...]...]
                                XR interfaces to create and their mapping to
                                underlying linux/pci interfaces
  --mgmt-interfaces linux:IF_NAME[,MG_FLAG[,...]][;linux:IF_NAME[...]...]
                                XR management interfaces to create and their
                                mapping to underlying linux interfaces (defaults
                                to a single interface mapped to eth0, pass ""
                                to prevent this)
  --first-boot-script FILE      Path to script to be run after all config has
                                been applied on the first boot
  --every-boot-script FILE      Path to script to be run after all config has
                                been applied on every boot
  --disk-limit LIMIT            Disk usage limit to impose (defaults to '6G')
  --ztp-enable                  Enable Zero Touch Provisioning (ZTP) to start
                                up after boot, by default ZTP is disabled
                                (cannot be used with IP snooping)
  --ztp-config FILE             Enable ZTP with custom ZTP ini configuration
  --args '<arg1> <arg2> ...'    Extra arguments to pass to '<ctr_mgr> run'

XRd Control Plane arguments:
  IF_TYPE := { linux }          Interface type
  IF_NAME := { * }              A linux interface name
  IF_FLAG := { xr_name | chksum | snoop_v[4|6] | snoop_v[4|6]_default_route }
                                Flags for interface configuration
  MG_FLAG := IF_FLAG            Flags for management interface configuration

XRd vRouter arguments:
  IF_TYPE := { pci | pci-range }
                                Interface type
  IF_NAME := { (IF_TYPE=pci)       BUS:SLOT.FUNC |
               (IF_TYPE=pci-range) lastN | firstN }
                                Either PCI address e.g. pci:00:09.0, or
                                selection of addresses e.g. pci-range:last4
  IF_FLAG := {}                 Flags for interface configuration
  MG_FLAG := { chksum | snoop_v[4|6] | snoop_v[4|6]_default_route }
                                Flags for management interface configuration
cisco@xrdcisco:~/xrd-tools/scripts$ 

```



## Using docker to launch XRd


### Load docker images downloaded from CCO
Before we begin, let's load the XRd images we downloaded from CCO into the local docker daemon.

The image tarballs we expanded in [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them) are dumped below.  

```bash
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

Loading the xrd-control-plane docker image first (we rename it explicitly to localhost/xrd-control-plane to easily differentiate the image from the xrd-vrouter image that we will load subsequently):  

```bash
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$ cd ../xrd-control-plane/
cisco@xrdcisco:~/images/xrd-control-plane$ docker load -i xrd-control-plane-container-x64.dockerv1.tgz
a42828b8fe58: Loading layer [==================================================>]  1.179GB/1.179GB
Loaded image: localhost/ios-xr:7.7.1
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ docker tag localhost/ios-xr:7.7.1 localhost/xrd-control-plane
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ docker rmi localhost/ios-xr:7.7.1
Untagged: localhost/ios-xr:7.7.1
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED       SIZE
localhost/xrd-control-plane   latest    dd8d741e50b2   4 weeks ago   1.15GB
cisco@xrdcisco:~/images/xrd-control-plane$
```

Similarly, let's load the xrd-vrouter docker image and rename it to localhost/xrd-vrouter

```bash
cisco@xrdcisco:~/images/xrd-control-plane$ 
cisco@xrdcisco:~/images/xrd-control-plane$ cd ../xrd-vrouter/
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$ docker load -i xrd-vrouter-container-x64.dockerv1.tgz
e97a8613578a: Loading layer [==================================================>]  1.225GB/1.225GB
Loaded image: localhost/ios-xr:7.7.1
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$ docker tag localhost/ios-xr:7.7.1 localhost/xrd-vrouter
cisco@xrdcisco:~/images/xrd-vrouter$ docker rmi localhost/ios-xr:7.7.1
Untagged: localhost/ios-xr:7.7.1
cisco@xrdcisco:~/images/xrd-control-plane$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED       SIZE
localhost/xrd-vrouter         latest    78632a9bbb1d   4 weeks ago   1.2GB
localhost/xrd-control-plane   latest    dd8d741e50b2   4 weeks ago   1.15GB
cisco@xrdcisco:~/images/xrd-vrouter$ 
cisco@xrdcisco:~/images/xrd-vrouter$ 
```


### Boot XRd control-plane image using launch-xrd


Before we try booting the image, let's use the `--dry-run` option in `launch-xrd` to dump the actual `docker run` command that would have been used in the background by the script:  


<div class="highlighter-rouge">
<pre class="highlight">
<code style="white-space: pre;">
cisco@xrdcisco:~/xrd-tools/scripts$<mark> ./launch-xrd --dry-run localhost/xrd-control-plane
docker run -it --rm --cap-drop all --cap-add AUDIT_WRITE --cap-add CHOWN --cap-add DAC_OVERRIDE --cap-add FOWNER --cap-add FSETID --cap-add KILL --cap-add MKNOD --cap-add NET_BIND_SERVICE --cap-add NET_RAW --cap-add SETFCAP --cap-add SETGID --cap-add SETUID --cap-add SETPCAP --cap-add SYS_CHROOT --cap-add IPC_LOCK --cap-add NET_ADMIN --cap-add SYS_ADMIN --cap-add SYS_NICE --cap-add SYS_PTRACE --cap-add SYS_RESOURCE --device /dev/fuse --device /dev/net/tun --security-opt apparmor=unconfined --security-opt label=disable -v /sys/fs/cgroup:/sys/fs/cgroup:ro --env XR_MGMT_INTERFACES=linux:eth0,chksum localhost/xrd-control-plane</mark>
</code>
</pre>
</div>


Structuring the output to better understand it:

```
docker run -it --rm \
--cap-drop all \
--cap-add AUDIT_WRITE --cap-add CHOWN \
--cap-add DAC_OVERRIDE --cap-add FOWNER \
--cap-add FSETID --cap-add KILL \
--cap-add MKNOD --cap-add NET_BIND_SERVICE \
--cap-add NET_RAW --cap-add SETFCAP \
--cap-add SETGID --cap-add SETUID \
--cap-add SETPCAP --cap-add SYS_CHROOT \
--cap-add IPC_LOCK --cap-add NET_ADMIN \
--cap-add SYS_ADMIN --cap-add SYS_NICE \
--cap-add SYS_PTRACE --cap-add SYS_RESOURCE \
--device /dev/fuse \
--device /dev/net/tun \
--security-opt apparmor=unconfined \
--security-opt label=disable \
-v /sys/fs/cgroup:/sys/fs/cgroup:ro \
--env
```






Part-5 of the XRd tutorials Series: [here]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-compose-control-plane-and-vrouter).
