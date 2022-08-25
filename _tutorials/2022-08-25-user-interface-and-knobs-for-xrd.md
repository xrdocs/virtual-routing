---
published: true
date: '2022-08-25 16:03 +0530'
title: User Interface and Knobs for XRd
author: Akshat Sharma
excerpt: >-
  Learn the different environment variables and knobs that are supported for XRd
  when used with container orchestrators like docker, docker-compose or k8s
tags:
  - iosxr
  - xrd-tutorial-series
  - virtual
  - container
  - xrd
  - vm
  - xrv9k
  - kubernetes
  - docker
  - linux
position: top
---

{% include base_path %}
{% include toc %}

* This is Part-4 of the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series).   
* Skip to Part-5 here: [XRd with Docker: Control-Plane and vRouter]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter).    
* Re-read Part-3 here: [Setting up the Host Environment to run XRd]({{base_path}}/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd/)


XRd boot is configured via environment variables passed to the container orchestrator. This page lists each way in which the boot of XRd can be configured.

**TIP**: The reading of environment variables uses fuzzy matching, so guessed variable names may error with spelling hints. As an example:
{: .notice--info}. 

```
 docker run --rm --privileged --env XR_VROUTER_DP_HUGEPAGE=1024 rebuild
WARNING: Unexpected XR env var XR_VROUTER_DP_HUGEPAGE, did you mean XR_VROUTER_DP_HUGEPAGE_MB?
ERROR: Not enough total HugePages free. Required: 3072M, Free: 2048M
ERROR: Platform init failed
XRd hit a critical error during initialization and has aborted launch.
```

## XRd Environment Variables
The tables in this section detail the full set of launch environment variables that XRd supports for configuring various behaviors. None of the variables are mandatory, and the default behavior when not specified is given too.

An advanced section covers environment variables that are only expected to be used under guidance by the engineering team to tweak advanced parameters.

### Common Variables
Variables that can be used across all XRd platforms.

| Variable               | Purpose                                                                                                                    | Contents                                                 | Default                |
|------------------------|----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|------------------------|
| XR_INTERFACES          | For specifying interfaces to be used as XR data interfaces                                                                 | See interface specification                              | No interfaces are used |
| XR_MGMT_INTERFACES     | For specifying interfaces to be used as XR management interfaces                                                           | See interface specification                              | No interfaces are used |
| XR_FIRST_BOOT_CONFIG   | For specifying a startup config file that should be used for first boot                                                    | Path to config file that has been mounted in by the user | None                   |
| XR_EVERY_BOOT_CONFIG   | For specifying a startup config to be used on every boot. This is ignored for first boot if the above env var is specified | Path to config file that has been mounted in by the user | None                   |
| XR_FIRST_BOOT_SCRIPT   | For specifying a startup script to be run on first boot                                                                    | Path to script that has been mounted in by the user      | None                   |
| XR_EVERY_BOOT_SCRIPT   | For specifying a startup script to be run on every boot. This is ignored for first boot if the above env var is specified  | Path to script file that has been mounted in by the user | None                   |
| XR_DISK_USAGE_LIMIT    | The disk usage limit which the disk cleanup will attempt to remain under by removing old files                             | A disk size value with units                             | 6G                     |
| XR_ZTP_ENABLE          | For enabling Zero Touch Provisioning (ZTP) at boot                                                                         | If set, 0 or 1                                           | ZTP is disabled        |
| XR_ZTP_ENABLE_WITH_INI | For enabling ZTP with a custom "ztp.ini" config file at boot                                                               | Path to mounted ztp.ini file                             | None                   |
| XR_BOOT_LOG_LEVEL      | (7.8.1) For controlling the level at which the XR boot logging starts being printed to the console                         | One of ERROR, WARNING, INFO or DEBUG                     | WARNING                |




### XRd vRouter Variables
Variables specific to the XRd vRouter platform:  


| Variable | XR_VROUTER_PCI_DRIVER                                    | XR_VROUTER_DP_HUGEPAGE_MB                           | XR_VROUTER_DP_CPUSET                                    | XR_VROUTER_CPUSET_AVOID                                                                                          | XR_VROUTER_DP_MAIN_CORE                           | XR_VROUTER_PCI_ERROR_VERBOSE                                                  |
|----------|----------------------------------------------------------|-----------------------------------------------------|---------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|-------------------------------------------------------------------------------|
| Purpose  | The driver to use for PCI interfaces that need rebinding | The number of MiB of hugepages to provide to XR     | The CPU set to be used for the dataplane packet threads | Avoid implicitly assigning any workloads to these CPU cores (they can still be used by explicit env var cpusets) | The CPU core to use for the dataplane main thread | Control printing of the full allow-list if an unsupported PCI device is given |
| Contents | One of igb_uio or vfio-pci                               | Integer value representing number of MiB (min 1024) | See CPU options                                         | See CPU options                                                                                                  | See CPU options                                   | 1 (boolean 'true')                                                            |
| Default  | vfio-pci                                                 | 3072                                                | See CPU options                                         | None                                                                                                             | See CPU options                                   | None                                                                          |
  
  
## Advanced Variables
These variables are not expected to be used in the mainline but are provided to allow finer grained control over certain parameters.

### XRd vRouter Variables  

| Variable | XR_VROUTER_DP_MAIN_TUNE                                                                                                                                     | XR_VROUTER_DP_CPUSET_WKXR_VROUTER_DP_CPUSET_TXXR_VROUTER_DP_CPUSET_RX   | XR_VROUTER_PCI_PERMIT_DEVICES                                                   |
|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| Purpose  | Dataplane main thread tuning settings, where tuned mode is where the dataplane main thread is tuned to be friendlier when sharing core with other workloads | The cpusets to use for the dataplane packet RX, TX and worker thread(s) | Comma-separated list of device type IDs to add to the PCI device type allowlist |
| Contents | See advanced CPU options                                                                                                                                    | See advanced CPU options                                                | See PCI interface details                                                       |
| Default  | 1                                                                                                                                                           | Automatic allocation using CPUs in the XR_VROUTER_DP_CPUSET             | None                                                                            |
  
  
## Interface Specification
The set of interfaces used by XRd, and their options, are specified in the XR_INTERFACES and XR_MGMT_INTERFACES environment variables. XR_INTERFACES is used to specify data ports, and XR_MGMT_INTERFACES is used to specify management ports.

They both follow the same format, consisting of a semi-colon (and optional whitespace) separated list of elements of the form:

<underlying interface type>:<underlying interface identifier>,<optional flags>

Where:

* **Underlying interface type** is one of:

  * linux: A linux interface identified by name. This option is applicable to data and management ports on XRd Control Plane, and to management ports on XRd vRouter.

  * pci: An interface identified by PCI address. Note this option is only applicable to data ports on XRd vRouter. Attempting to use this option on XRd Control Plane or in the XR_MGMT_INTERFACES environment variable will result in an error.

  * pci-range: A range in the ordered list of available and supported PCI interfaces discovered at boot. Only supported for XR_INTERFACES on XRd vRouter. No other PCI interfaces may be specified (by pci/pci-range) in addition to this type.

* **Underlying interface identifier** is an identifier corresponding to the specified interface type, respectively:

  * The linux interface name for interfaces with type "linux"

  * The pci address for interfaces with type "pci"

  * A range description using the keywords "first" or "last" followed by a positive integer N

Linux interface names must be decodable using the URL encoding scheme. This does not affect alphanumeric characters (i.e. letters or numbers), but for example "=" and ";" become "%3D" and "%3B" respectively when URL encoded.

**NOTE**:
The decoded interface name must not include whitespace, and there is a short-term limitation on XRd Control Plane that the decoded interface name must not include commas or colons.
{: .notice-info}. 
  
* **Optional flag** are comma separated keywords (not supported for "pci-range" type):

  * xr_name=<XR interface name> to specify an XR interface name to represent this interface

    * Fully qualified name, with support for both short and long XR interface types (ie Gi, GigabitEthernet, Mg, MgmtEth)

    * For the R/S/I/P section, only the port number may be customized - R/S/I must be 0/0/0 for data ports and 0/RP0/CPU0 for management ports

  * chksum: indicate that TCP/UDP checksums need to be calculated by XRd for ingress packets to counteract checksum offload. This is only supported for linux interfaces.

  * snoop_{v4,v6}: indicate that this interface's address (IPv4 or IPv6 as appropriate) should be snooped and applied as XR config. It is possible to specify any combination of these flags. This is only supported for linux interfaces on XRd Control Plane and management interfaces on XRd vRouter. These flags may not be used with ZTP enabled.

  * snoop_{v4,v6}_default_route: indicate that the IPv4 or IPv6 default route for this interface should be snooped. If either/both of these flags are specified, the corresponding snoop_{v4,v6} flags must also be specified, or this will result in an error. The flags can only be specified for at most one interface in total. This is only supported for linux interfaces on XRd Control Plane and management interfaces on XRd vRouter.

If the xr_name flag is not specified, the next available XR port number is used (starting at 0, and after taking into account ports where xr_name is specified). This is used in conjunction with the following interface name and R/S/I segment:

* For XRd Control Plane:

  * GigE0/0/0/X if the interface is in XR_INTERFACES

  * MgmtEth0/RP0/CPU0/X if the interface is in XR_MGMT_INTERFACES

* For XRd vRouter:

  * Name depends on detected underlying interface speed if the interface is in XR_INTERFACES

  * MgmtEth0/RP0/CPU0/0 if the interface is in XR_MGMT_INTERFACES, as only one management interface is currently supported

The below tables summarize the support for all supported "Interface type (XRd platform)" permutations:

### XR_INTERFACES:  
  

| linux (XRd Control Plane) | Identifier           | xr_name | chksum | snoop_{v4,v6} | snoop_{v4,v6}_default_route |
|---------------------------|----------------------|---------|--------|---------------|-----------------------------|
| pci (XRd vRouter)         | Linux interface name | ✓       | ✓      | ✓             | ✓                           |
| pci-range (XRd vRouter)   | PCI address          | X       | X      | X             | X                           |
  
  
### XR_MGMT_INTERFACES:  
  

| linux (XRd Control Plane) | Identifier           | xr_name | chksum | snoop_{v4,v6} | snoop_{v4,v6}_default_route |
|---------------------------|----------------------|---------|--------|---------------|-----------------------------|
| linux (XRd vRouter)       | Linux interface name | ✓       | ✓      | ✓             | ✓                           |

  
  
  
## Examples:  
  
```  
XR_INTERFACES="linux:eth0;linux:eth1"
XR_INTERFACES="linux:eth0,snoop_v4,xr_name=GigabitEthernet0/0/0/1;linux:eth1,chksum,xr_name=GigE0/0/0/0"
XR_INTERFACES="pci:00:08.1;pci:00:09.0"
XR_MGMT_INTERFACES="linux:eth0,chksum"
XR_INTERFACES="pci:00:08.0;\
               pci:00:09.0"
XR_INTERFACES="linux:eth0%3B1,xr_name=Gi0/0/0/10"
XR_INTERFACES="pci-range:last4"
```

The following gives an example specification of XR_INTERFACES and XR_MGMT_INTERFACES to 'docker run':

```
docker run <other args> \
  --env XR_MGMT_INTERFACES="linux:eth0,xr_name=Mg0/RP0/CPU0/0,snoop_v6,snoop_v6_default_route" \
  --env XR_INTERFACES="linux:eth1,xr_name=GigE0/RP0/CPU0/0" \
  <image name>
```
    
## PCI Interface Details  
    
On XRd vRouter, the XR_INTERFACES environment variable is composed of either:

* A semicolon separated list of PCI addresses corresponding to single interfaces, like:  
    
  ```
  pci:<optional domain>:<bus>:<slot>.<func>   
  ```

  * Domain can be omitted, and when not specified 0000 is assumed.

  * Note: only domain ID 0000 is currently supported.

* A single range of unspecified PCI addresses of length N, like:

  ```  
  pci-range:{first,last}N
  ```
    
  * Addresses are taken from a discovered list of available and supported devices (of the networking class and on the allowed device type list)

  * Keyword "first" selects from the start of the numerically ordered list and "last" selects from the end

  * The boot fails if N is greater than the list of available and supported devices

  * No additional flags are supported in this case

The `XR_VROUTER_PCI_PERMIT_DEVICES` variable is a comma-separated list that can be set with additional PCI device types to permit in addition to the allowlist. This prevents needing to rebuild an image if a device type is missing.

If the `XR_VROUTER_PCI_ERROR_VERBOSE` variable is set, the assertion of a device type being on the allowlist prints the full allowlist on failure.

#### Examples:  

```
docker run <other args> \
  --env XR_INTERFACES="pci:00:03.0;pci:00:04.0" \
  <image name>
```
    
A container orchestrator may take control of the first available address and the user may not know what addresses are available before boot. The user can discover addresses from the end of the list to avoid conflict with the orchestrator:
    
```
docker run <other args> \
  --env XR_INTERFACES="pci-range:last3" \
  <image name>
```
    
By default, if not already bound to one of the supported drivers then XRd will rebind the interfaces to the default driver. If the user wishes to control which driver the default is, then they can use the XR_VROUTER_PCI_DRIVER environment variable (see section 4.1.1 for more details on interface drivers):

```
docker run <other args> \
  --env XR_VROUTER_PCI_DRIVER="igb_uio" \
  <image name>
```
    
   
## CPU Options
By default, XRd vRouter selects a single CPU core for the packet thread, then uses the remainder of the available CPU cores for control-plane usage and the dataplane main thread.

There are 3 environment variables that can be used to control this behavior:

* `XR_VROUTER_DP_CPUSET` – use this cpuset for the dataplane threads. If more than 1 cpu then multiple rx, tx and worker threads are created to utilize the threads assigned. Control-plane processes will be assigned to available CPU cores NOT in this cpuset.

* `XR_VROUTER_DP_MAIN_CORE` – use this core for the dataplane main thread. The dataplane main thread gets assigned to the last control-plane CPU by default, this envvar allows this behavior to be changed.

* `XR_VROUTER_CPUSET_AVOID` – avoid implicitly assigning any workloads to these CPUs when automatically picking control-plane and dataplane CPUs. The user can still use these CPUs when explicitly assigning CPUs via the other envvars here.

Examples when this might be used:

* when running in a hyperthreading setup and want to ensure nothing uses sibling core(s) for core(s) used by dataplane packet threads

* when wanting to give the dataplane main thread an exclusive core.

The CPUSET variables take as their arguments a cpuset (comma separated list of cpu core indices or ranges of indices).

Docker and Kubernetes both allow restrictions to be placed on the CPU resource that the container is allowed to use – such as via the --cpuset-cpus Docker 'run' argument. If these are specified to the container orchestrator then the CPUSET variables described within this section must be within that restricted set.

Examples:  
    
```    
docker run <other args> \
  --env XR_VROUTER_DP_CPUSET="0-1" \
  <image name>
```
    
Example of 4 logical core HT setup using envvars to ensure dataplane thread gets dedicated physical CPU:

```
docker run <other args> \
  --cpuset-cpus 0-3
  --env XR_VROUTER_DP_CPUSET="2" \
  --env XR_VROUTER_CPUSET_AVOID="3" \
  <image name>
```
    
This will result in assigning cores 0-1 to control-plane, 1 the dataplane main thread and a single packet thread to 2 and nothing to 3

Example of giving the dataplane main thread an exclusive core:

```
docker run <other args> \
  --cpuset-cpus 0-2
  --env XR_VROUTER_DP_MAIN_CORE="1" \
  --env XR_VROUTER_CPUSET_AVOID="1" \
  <image name>
```
    
This will result in assigning core 0 to control-plane, 1 to the dataplane main thread and a single packet thread to 2

Advanced CPU Options
This section documents in detail the advanced CPU options.

XR_VROUTER_DP_MAIN_TUNE – this allows the dataplane main thread's tuning parameters to be fine tuned. These tuning parameters allow the dataplane main thread to share a CPU core with other workloads. It supports the following values (where '1' is the default value when not specified).

0 - turn off dataplane main thread tuned mode

1 - use dataplane main thread tuned mode with default settings: 1000,50,500,20

sleep us,pkt processing us,msg processing us,max punt pkts - use dataplane main thread tuned mode with these specific settings

XR_VROUTER_DP_CPUSET_WK, XR_VROUTER_DP_CPUSET_TX, XR_VROUTER_DP_CPUSET_RX – this allows fine grained control over the number of each of the packet thread types and which CPUs they are allocated to.

If one of the XR_VROUTER_DP_CPUSET_XXX options is specified, then the lack of the others implies none of those threads.

Must specify at least 1 worker and tx thread.

Cannot overlap with each other or the dataplane main core.

Cannot be specified at same time as XR_VROUTER_DP_CPUSET.


Part-5 of the XRd tutorials Series here: [XRd with Docker: Control-Plane and vRouter]({{base_path}}/tutorials/2022-08-23-xrd-with-docker-control-plane-and-vrouter)




