---
published: true
date: '2023-05-02 16:43 -0700'
title: XRd on Openshift
author: Taran Deshpande
position: hidden
---
# Introduction

# Tuning the Worker Node
In our previous [tutorial](https://xrdocs.io/virtual-routing/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd/), we outlined the host requirements of running XRd and configured a Ubuntu 20.04 host machine. To deploy XRd on Openshift, each worker node must meet these host requirements. 
## Host Check
The [host check script](https://github.com/ios-xr/xrd-tools/blob/main/scripts/host-check) will run on a pod instead of as a script directly on the worker node.

<p class="codeblock-label">host check Dockerfile</p>
```bash
FROM registry.access.redhat.com/ubi8/python-39

USER 0
ADD host-check .
RUN chown -R 1001:0 ./
USER 1001

CMD python3 host-check
```

<p class="codeblock-label">host_check_pod.yaml</p>
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: xrd-host-check
  namespace: <project namespace>
spec:
  containers:
  - image: quay.io/rh_ee_ttracey/xrd/host-check:v1.0
    imagePullPolicy: IfNotPresent
    name: host-check
    resources:
      requests:
        memory: 2Gi
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /lib/modules
      name: hostpath-lib-modules
      readOnly: true
  priority: 0
  # serviceAccountName: iosxr 	TODO: is this required?
  hostNetwork: true
  restartPolicy: Never
  volumes:
    - name: hostpath-lib-modules
      hostPath:
          path: /lib/modules
```
The output of the host check script can be viewed by checking the logs of `xrd-host-check` pod.

```bash
tadeshpa@TADESHPA-M-F92B ~/openshift [1]> oc logs xrd-host-check
==============================
Platform checks
==============================

base checks
-----------------------
PASS -- CPU architecture (x86_64)
PASS -- CPU cores (80)
PASS -- Kernel version (4.18)
PASS -- Base kernel modules
        Installed module(s): dummy, nf_tables
PASS -- Cgroups (v1)
PASS -- Inotify max user instances
        64000 - this is expected to be sufficient for 16 XRd instance(s).
PASS -- Inotify max user watches
        65536 - this is expected to be sufficient for 16 XRd instance(s).
PASS -- Socket kernel parameters (valid settings)
PASS -- UDP kernel parameters (valid settings)
INFO -- Core pattern (core files managed by the host)
PASS -- ASLR (full randomization)
INFO -- Linux Security Modules (No LSMs are enabled)

xrd-control-plane checks
-----------------------
PASS -- RAM
        Available RAM is 279.1 GiB.
        This is estimated to be sufficient for 139 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.

xrd-vrouter checks
-----------------------
PASS -- CPU extensions (sse4_1, sse4_2, ssse3)
PASS -- RAM
        Available RAM is 279.1 GiB.
        This is estimated to be sufficient for 55 XRd instance(s), although memory
        usage depends on the running configuration.
        Note that any swap that may be available is not included.
PASS -- Hugepages (52 x 1GiB)
FAIL -- Interface kernel driver
        None of the expected PCI drivers are loaded.
        The following PCI drivers are installed but not loaded: vfio-pci.
        Run 'modprobe <pci driver>' to load a driver.
SKIP -- IOMMU
        Skipped due to failed checks: Interface kernel driver
PASS -- Shared memory pages max size (17179869184.0 GiB)

==================================================================
XR platforms supported: xrd-control-plane
XR platforms NOT supported: xrd-vrouter
==================================================================
```

## Machine Config

We will use the [Machine Config Operator](https://docs.openshift.com/container-platform/4.12/post_installation_configuration/machine-configuration-tasks.html) to set the Inotify max user watches and Inotify max user instances settings. 

<p class="codeblock-label">sysctl_mc.yaml</p>
```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-sysctl-inotify-override-iosxr
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 3.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
       - contents:
           source: data:,%0Afs.inotify.max_user_watches%20%3D%2065536%0Afs.inotify.max_user_instances%20%3D%2064000%0A
         mode: 420
         overwrite: true
         path: /etc/sysctl.d/inotify.conf
```

Apply the configuration with: `oc apply -f sysctl_mc.yaml`
## TuneD
The [node tuning operator](https://docs.openshift.com/container-platform/4.12/scalability_and_performance/using-node-tuning-operator.html) sets up some kernel parameters and tuning options to help XRd achieve high performance.

<p class="codeblock-label">tuned.yaml</p>
```yaml
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: sysctl-updates-iosxr
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - data: |
      [main]
      summary=A custom profile for Cisco xrd
      include=openshift-node-performance-iosxr-performanceprofile
      [sysctl]
      net.ipv4.ip_local_port_range="1024 65535"
      net.ipv4.tcp_tw_reuse=1
      fs.inotify.max_user_instances=64000
      fs.inotify.max_user_watches=64000
      kernel.randomize_va_space=2
      net.core.rmem_max=67108864
      net.core.wmem_max=67108864
      net.core.rmem_default=67108864
      net.core.wmem_default=67108864
      net.core.netdev_max_backlog=300000
      net.core.optmem_max=67108864
      net.ipv4.udp_mem="1124736 10000000 67108864"
    name: cisco-xrd
  recommend:
  - machineConfigLabels:
      machineconfiguration.openshift.io/role: master
    priority: 10
    profile: cisco-xrd
```
    
Apply the configuration with: `oc apply -f tuned.yaml`
## Performance Profile
Create a Performance Profile to set the number of desired Hugepages and reserved CPUs. HugePages of size 1GiB must be enabled with a total of 3GiB of available HugePages RAM for each XRd vRouter. Remember to enable Hugepages for each NUMA node that will be running XRd.

<p class="codeblock-label">pao.yaml</p>
```yaml
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
  name: iosxr-performanceprofile
spec:
  additionalKernelArgs:
  cpu:
    isolated: 4-39
    reserved: 0-3
  hugepages:
    defaultHugepagesSize: 1G
    pages:
      - count: 32
        node: 0
        size: 1G
      - count: 32
        node: 1
        size: 1G
  nodeSelector:
    node-role.kubernetes.io/master: ''
  realTimeKernel:
    enabled: false
```

Apply the configuration with: `oc apply -f pao.yaml`
# Deploy XRd Control Plane
