---
published: true
date: '2022-08-22 18:54 +0530'
title: 'Helper Tools to get started with XRd '
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - kubernetes
  - virtual
  - xrd
  - xrv9k
  - docker
  - container
  - vm
  - xrd-tutorial-series
excerpt: >-
  Helper scripts and tools to launch Cisco XRd platform in containerized network
  environments.
position: hidden
---

{% include base_path %}
{% include toc %}

This is Part-2 of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series). Skip to Part-3 [here]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd).   

## XRd Helper Tools

In [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them/) of the XRd Tutorial Series, we learnt how to download and verify XRd images from software.cisco.com for both the form factors:
1. XRd Control-Plane
2. XRd vRouter

In this tutorial, let's introduce a few helper tools that have been made available publicly to ease the process of testing and deploying XRd locally.

A set of scripts, samples and templates are published to Github here:
>[https://github.com/ios-xr/xrd-tools](https://github.com/ios-xr/xrd-tools).


We will utilize these tools throughout the XRd Tutorials Series as we take our first steps with XRd.


## Taking Stock of the XRd-Tools repository

Let's clone the git repo:

```bash
cisco@xrdcisco:~$ git clone https://github.com/ios-xr/xrd-tools.git
Cloning into 'xrd-tools'...
remote: Enumerating objects: 69, done.
remote: Counting objects: 100% (69/69), done.
remote: Compressing objects: 100% (43/43), done.
remote: Total 69 (delta 27), reused 61 (delta 24), pack-reused 0
Unpacking objects: 100% (69/69), 84.37 KiB | 2.01 MiB/s, done.
cisco@xrdcisco:~$ 
cisco@xrdcisco:~$ 

```  


Dumping the file structure of the repository:

```bash
cisco@xrdcisco:~$ tree xrd-tools/
xrd-tools/
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── docs
│   └── version-compatibility.md
├── mypy.ini
├── pylintrc
├── pyproject.toml
├── requirements.txt
├── samples
│   └── xr_compose_topos
│       ├── bgp-ospf-triangle
│       │   ├── docker-compose.xr.yml
│       │   ├── topo-diagram.png
│       │   ├── xrd1_xrconf.cfg
│       │   ├── xrd2_xrconf.cfg
│       │   └── xrd3_xrconf.cfg
│       ├── segment-routing
│       │   ├── docker-compose.xr.yml
│       │   ├── xrd-1-startup.cfg
│       │   ├── xrd-2-startup.cfg
│       │   ├── xrd-3-startup.cfg
│       │   ├── xrd-4-startup.cfg
│       │   ├── xrd-5-startup.cfg
│       │   ├── xrd-6-startup.cfg
│       │   ├── xrd-7-startup.cfg
│       │   └── xrd-8-startup.cfg
│       └── simple-bgp
│           ├── docker-compose.xr.yml
│           ├── xrd-1_xrconf.cfg
│           └── xrd-2_xrconf.cfg
├── scripts
│   ├── apply-bugfixes
│   ├── host-check
│   ├── launch-xrd
│   └── xr-compose
├── templates
│   ├── docker-compose.template.xr.yml
│   └── launch-xrd-macvlan.template
└── tests
    ├── __init__.py
    ├── test_host_check.py
    └── utils.py

9 directories, 35 files
cisco@xrdcisco:~$ 


```

The `scripts/` and `samples/` directories are of particular interest to an end user as they help in setting up the host environment and abstracting out the various knobs required in docker-compose files to spin up working XRd environments.





## System Dependencies

XRd is only able to run on Linux, therefore the scripts provided here are also targeting Linux.

The scripts are implemented in bash and python3, which must therefore be found on PATH. All active versions of python3 are supported.


The `requirements.txt` file can be used to install the entire set of dependencies required for basic xrd-tools scripts as well as for the py-test scripts to succeed.

However, to enable the use of the scripts in the `scripts/` folder, we must ensure that 
docker-compose (v1) is available in PATH and the PyYAML python package is installed (e.g. in an active virtual environment).


## Exploring the Scripts

We will utilize the following scripts and discuss their nuances in detail in subsequent parts of the [XRd Tutorial Series]({{base_path}}/tags/#xrd-tutorial-series) usage of the scripts by passing --help.

host-check - Check the host is set up correctly for running XRd containers.
launch-xrd - Launch a single XRd container, or use --dry-run to see the args required.
xr-compose - Launch a topology of XRd containers (wraps docker-compose).
apply-bugfixes - Create a new XRd image with bugfixes installed on top of a base image.




Part-3 of the XRd tutorials Series: [here]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd).
