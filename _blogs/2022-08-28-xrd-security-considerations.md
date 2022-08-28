---
published: true
date: '2022-08-28 22:33 +0530'
title: 'XRd: Security Considerations'
author: Akshat Sharma
excerpt: >-
  A breakdown of the security considerations that apply to XRd usage with
  container orchestrators in production environments.
tags:
  - iosxr
  - linux
  - xrd
  - docker
  - vm
  - container
  - kubernetes
  - virtual
  - security
position: top
---
{% include base_path %}
{% include toc %}



## Security Considerations  

The [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series) walks the user through the steps to set up a host and run XRd but there are additional considerations pertaining to security. These cover both ensuring that XRd has sufficient privileges to run whilst also securing the host and XRd within it.
In this blog, we bring up some security considerations to be kept in mind and implemented when running XRd. Explicitly, we do not provide instructions on how an end user should secure their host - general system administration best practices should be used for that - but the blog highlights specific XRd considerations.

As a container-based platform, XRd does not have control over the host on which it runs; the host is the responsibility of the end user. This makes XRd different from other IOS-XR platforms - especially hardware-based platforms - which changes the security considerations of the end user. The user not only has to care about the security of the host environment but some security features of other platforms are simply not available. Whilst this is similar to VM-based platforms (such as XRv9k), the practical impact of the considerations for containers are different and generally more complex.


## Overview
The responsibility for securing the XRd container image within its runtime environment lies with the end user. Once the XRd image archive has been validated and loaded into a repository then XRd (or Cisco) cannot guarantee the security of the environment within which it is used.

The container landscape is evolving rapidly and when running XRd one must be familiar with the various technologies and keep up to date with the latest developments.
At a high-level the considerations encompass securing the host on which XRd is running, applying security principles to the container orchestrator and runtime when instantiating XRd, and managing the privileged users on the system.

## Primary Considerations
This section discusses items that must be considered regardless of how XRd is being used.

### Host system (Primary)
The requirements to secure the host can vary significantly depending on the individual host and wider environment.

#### Linux Kernel Security Policies

**AppArmor**:
Support for AppArmor in Kubernetes appears to still be in beta. Progress towards general availability seems to have stalled with "out-of-tree enhancements" offered instead. As such, AppArmor is not officially supported for XRd and it is up to the end user to configure a profile and enable it on the node hosts.

By default, if AppArmor is enabled and running on the host, launching Docker without the security option results in a profile docker-default being created and applied. To avoid this the option `--security-opt apparmor=unconfined` is used. Kubernetes by default does not apply a profile to launched pods. The command aa-status (run on the host) shows which profiles are currently enabled and applied to which profiles, and can be used to confirm that XRd is not being run with AppArmor protection.

**SELinux**:
In production use cases, it is not supported to run XRd in non-privileged mode. In privileged mode, all SELinux checks are bypassed, so therefore it is not supported to secure XRd with SELinux. 

On hosts with SELinux enabled, it is possible to run XRd in such a way that bypasses checks, but allowing other workloads to be constrained by SELinux, by passing the `--security-opt="label=disable"` option to docker run, or by using the `--privileged` option.

**Root of Trust**
As a software solution, XRd is unable to establish a Root of Trust anchored in the hardware (for example, via a TPM).
The end user is responsible for building and validating this trust chain and XRd has an implicit trust of the host.

The XRd container archive is signed with a detached signature that is distributed alongside the image allowing it to be validated by the end user - we show this process end-to-end in [Part-1]({{base_path}}/tutorials/2022-08-22-xrd-images-where-can-one-get-them/) of the XRd Tutorial Series.

**Resource Exhaustion**
If the resources on the host are exhausted such that XRd is starved of (for example) memory or inotify instances by other applications on the host then this could cause a disruption or denial of service.
The end user is responsible for ensuring that XRd is sufficiently resourced at all times on the host.

A single XRd container requires a minimum of 2000 inotify user instances/watches; it is recommended to 'set and forget' the limits on the host to be much higher (such as 128,000+).


### Container Orchestrator & Runtime (Primary)  

The container orchestrator and runtime obviously has a pivotal role in security as the gateway between the host and XRd.

The ability to run containers as a non-privileged user provides a significant improvement in the security posture of containers. However, this is currently not an option for XRd.

#### Container Privileges
XRd requires various Linux kernel capabilities that encompass the default Docker capabilities as well as a number of additional capabilities.
It is recommended to drop default privileges and explicitly specify all capabilities when starting a container so that any changes in the default set do not cause XRd to receive unnecessary privileges (or lose necessary privileges).
For example, docker run ... --cap-drop all --cap-add <CAP_1> --cap-add <CAP_2>.

This is the default behavior if using the `launch-xrd` or `xr-compose` tools (described in the [XRd tutorials Series]({{base_path}}/tags/#xrd-tutorial-series)).

**WARNING**: It is recommended to specify the container's capabilities and dependencies explicitly rather than using the --privileged option, as this ensures the container's privileges are kept to the minimum required. Note that for running in Kubernetes it is currently required to use privileged mode due to the need to mount devices.
{: .notice--warning}

The following are the default Docker capabilities (and must be explicitly specified if using --cap-drop all):

CAP_AUDIT_WRITE
CAP_CHOWN
CAP_DAC_OVERRIDE
CAP_FOWNER
CAP_FSETID
CAP_KILL
CAP_MKNOD
CAP_NET_BIND_SERVICE
CAP_NET_RAW
CAP_SETFCAP
CAP_SETGID
CAP_SETPCAP
CAP_SETUID
CAP_SYS_CHROOT
In addition to the default capabilities, all XRd platforms require the following capabilites:

| Capability       | Reason                                                          |
|------------------|-----------------------------------------------------------------|
| CAP_IPC_LOCK     | Required to use mlock                                           |
| CAP_NET_ADMIN    | Required for interface creation, and routing table modification |
| CAP_SYS_ADMIN    | Required for Filesystem in USErspace (FUSE) use                 |
| CAP_SYS_NICE     | Required to set process priorities, e.g. packet processing      |
| CAP_SYS_PTRACE   | Required during core production                                 |
| CAP_SYS_RESOURCE | Required for mqueues                                            |


The XRd vRouter platform also requires the following capabilities:

| Capability | Reason                                    |
|------------|-------------------------------------------|
| SYS_RAWIO  | Required for hardware I/O port operations |
  
  
The required list contains powerful capabilities - future work aims to reduce this list.

Secondary Considerations
The following considerations may not apply in the lab use case, where it is expected that the host is isolated. They would likely apply in a production deployment scenario and serve as a starting point for moving beyond the basic lab usage.

Host system (Secondary)
ASLR
Address space layout randomization (ASLR) is a security feature involved in preventing exploitation of memory corruption vulnerabilities by randomly arranging the address space of a process.
It requires that processes be compiled with position-independent executable code and the ASLR kernel module be enabled.

XRd is compiled position-independent and it is recommended that the ASLR kernel module be enabled to benefit from this protection.

Privileged Users
Privileged users on the host have complete control to do anything on the host, either maliciously or accidentally.
Obviously, a privileged user may modify any of the configuration covering considerations elsewhere on this page but they may also execute process in the container namespace without going through AAA. For example, a user may execute XRd processes without logging in.

Care must be taken to manage privileged access to the host.

Network Filtering
XRd only has basic ACL capabilities and does not provide any DDoS protection.
This must be provided by the surrounding environment and will be dependent on the specific environment. For example, the wider orchestration environment may provide this protection, or Netfilter (nftables/iptables) rules could be used to fulfil the requirements.

Container Orchestrator & Runtime (Secondary)
Container Resources
Docker does not constrain the memory available to a container by default and hence it may be appropriate that resource constraints are used to limit the container resources to prevent the host's memory from being exhausted.
The appropriate value to set depends on the specific usage but a minimum of 2GB is recommended, and 16GB or more may be required for running high-scale features.
Similarly, Docker does not constrain a container's access to the CPU by default but runtime options may be used to limit CPU usage; XRd requires at least one CPU core.

The launch-xrd and xr-compose tools do not constrain any resources by default.
If running a single instance then launch-xrd --dry-run can be used to see the base command that can then be supplemented with the necessary runtime options.
If running multiple instances with xr-compose then refer to the Compose Specification for how to specify resource constraints in the input YAML file.

Mounts
There are various security considerations that apply to storage mounted into the container.
The mandatory and optional mounts can be understood with the help of the launch-xrd help (--help) and dry run (--dry-run).

Mounts should be limited to the minimum required for XRd to run.
The end user is responsible for encrypting any persistent storage volume mounted into XRd.
It is recommended that any startup config file is read-only and is mounted to /etc/xrd/.
By default, mounts are not bounded in size and hence the host may be vulnerable to exhaustion - this covers both disk storage and memory (such as with tmpfs).
Explicitly:

Docker local volume mounts are limited only by the size of the filesystem on which the volume is mounted.
XRd has a feature that limits disk usage of the persistent storage to 6GiB (default) that may be configured with the --env XR_DISK_USAGE_LIMIT=<limit> to docker run (or the --disk-limit <limit> option to launch-xrd).
Additionally, the directory that volumes are stored on could be configured to be a separate filesystem from the rest of the host.
Alternative Docker volume drivers are out of scope of these considerations but could be used as a solution.
By default, the /run tmpfs is bounded to half the total memory of the host, regardless of the memory allocated to the container (see above).
The size of tmpfs can be constrained with the tmpfs-size option when specifying the mount.
The expected size of /run is expected to be small and 10% of the container's memory should be sufficient.
There are further considerations that XRd is currently not able to meet and hence must not be enacted:

XRd does not support mounting the root filesystem as read-only - explicitly, do not specify --read-only to docker run.
It would be preferable to mount persistent storage with the nodev,noexec,nosuid mount options but this is not currently supported.
Terminals & Logging
Docker will capture the output from stdout and stderr of the default PTY attached to the container and record it in the docker logs.
This may result in sensitive information - such as configuration or secrets - being logged in plaintext on the host.

If this is a concern then: either SSH should be used to connect to XRd; or the Docker logging driver may be disabled. It is not supported to attach additional terminals to XRd via docker exec.

Cgroup Filesystem Setup
Before 7.8.1 it was required to pass the host's cgroup filesystem through to the container to allow systemd to run within Docker in non-privileged mode. This gives the container full access to the host's cgroup filesystem, which is undesirable, and so should not be done now that it is no longer required.

The --cgroupns=private option is recommended to isolate the container further.

See Cgroup Filesystem Complexities for more details.

