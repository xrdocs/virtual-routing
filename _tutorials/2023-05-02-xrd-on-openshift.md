---
published: true
date: '2023-05-02 16:43 -0700'
title: XRd on Openshift
author: Taran Deshpande
position: hidden
---
# Introduction

# Tuning the Worker Node
In our previous [tutorial](https://xrdocs.io/virtual-routing/tutorials/2022-08-22-setting-up-host-environment-to-run-xrd/), we outlined the host requirements of running XRd and configured a Ubuntu 20.04 host machine. To deploy XRd on Openshift, each worker node must meet these host requirements. In this tutorial, I've done a single-node install on a UCS C220 M5, so there is only one worker node.

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
Create a Performance Profile to set the number of desired Hugepages as well as reserved and isolated CPUs. HugePages of size 1GiB must be enabled with a total of 3GiB of available HugePages RAM for each XRd vRouter. Remember to enable Hugepages for each NUMA node that will be running XRd. The isolated CPUs are ones that will be available to be pinned to specific XRd workloads.

<p class="codeblock-label">pao.yaml</p>
```yaml
apiVersion: performance.openshift.io/v2
kind: PerformanceProfile
metadata:
  name: iosxr-performanceprofile
spec:
  additionalKernelArgs:
  cpu:
    isolated: 4-79
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

## Load PCI driver
The vfio-pci driver must be loaded for the XRd vRouter to use PCI passthrough. Creating just one VF will load the vfio-pci driver on the worker node. In this example, we have an Intel X710 NIC, and this is reflected in the nicSelector field with relevant vendor, deviceID, pfNames, and rootDevices values. 
<p class="codeblock-label">intel-dpdk-node-policy.yaml</p>
```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: intel-dpdk-node-policy
  namespace: openshift-sriov-network-operator
spec:
  resourceName: intelnics
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
  priority: 0
  numVfs: 1
  nicSelector:
    vendor: "8086"
    deviceID: "1572"
    pfNames: ["ens1f1"]
    rootDevices: ["0000:5e:00.1"]
  deviceType: vfio-pci 
  ```
  
  We will create this NetworkNode Policy with: `oc create -f intel-dpdk-node-policy.yaml`
  
# Deploy XRd vRouter

## Add Helm Repository
[Helm](helm.sh) is a package manager for kubernetes, and there is a public [helm repo](https://ios-xr.github.io/xrd-helm/) for XRd, with [charts](https://github.com/ios-xr/xrd-helm/tree/main/charts) that can deploy a single instance of either the XRd Control-Plane or vRouter. The [value files](https://github.com/ios-xr/xrd-helm/blob/main/charts/xrd-vrouter/values.yaml) present in the repo document all possible settings that can be configured when deploying XRd on K8s.

To add the helm repo:
```
tadeshpa@TADESHPA-M-F92B ~/openshift> helm repo add xrd https://ios-xr.github.io/xrd-helm
```

And now we can see the charts that under the xrd repo:
```
tadeshpa@TADESHPA-M-F92B ~/openshift> helm search repo xrd/
NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                  
xrd/xrd-common          1.0.2                           Common helpers for Cisco IOS-XR XRd platforms
xrd/xrd-control-plane   1.0.2                           Cisco IOS-XR XRd Control Plane platform      
xrd/xrd-vrouter         1.0.2                           Cisco IOS-XR XRd vRouter platform
```

## Deploy a single instance of XRd vRouter

Now let's deploy a single instance of the XRd vRouter attached to one PCI interface. Make sure the XRd vrouter is hosted on a container repository and accessible from the host. We are binding it to the NIC with PCI address 5e:00.1, and we are pinning cpus 5 and 6 for XRd. We will create our own custom values file that describes this.

values.yaml
```yaml
config:
  ascii: |
    hostname xrd1
  username: cisco
  password: cisco123
  scriptEveryBoot: false
  ztpEnable: false
cpu:
  cpuset: 5-6
hostNetwork: false
image:
  pullPolicy: Always
  repository: <your-container-repository>
  tag: <your-container-tag>
interfaces:
- config:
    device: 5e:00.1
  type: pci
```

Now to launch:
```
tadeshpa@TADESHPA-M-F92B ~/openshift [1]> helm install xrd-single xrd/xrd-vrouter -f values.yaml
NAME: xrd-single
LAST DEPLOYED: Wed Jun 14 17:14:52 2023
NAMESPACE: openshift
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have installed XRd vRouter version 7.8.1.
```

After a few mins we can see that our pod has launched:
```
tadeshpa@TADESHPA-M-F92B ~/openshift> oc get pods
NAME                               READY   STATUS              RESTARTS   AGE
xrd-single-xrd-vrouter-0           1/1     Running             0          2m4s
```

And we can also view the syslogs:
```
tadeshpa@TADESHPA-M-F92B ~/openshift> oc logs xrd-single-xrd-vrouter-0
CPU assignments
  available cpuset: node0 0-19,40-59, node1 20-39,60-79
  control-plane cpuset: 5
  dataplane main cpuset: 5 (tune enabled)
  dataplane packet cpuset: 6 (rx:- tx:- wk:6)
Hugepage assignment (per NUMA node): node0 3072M, node1 0M
Using interfaces: pci:5e:00.1
systemd 230 running in system mode. (+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP -LIBCRYPTSETUP -GCRYPT -GNUTLS +ACL +XZ -LZ4 -SECCOMP +BLKID -ELFUTILS +KMOD -IDN)
Detected virtualization container-other.
Detected architecture x86-64.

Welcome to Cisco XR (Base Distro SELinux and CGL) 9.0.0.26!
```

To exec into the xr shell of the pod:
```
tadeshpa@TADESHPA-M-F92B ~/openshift> oc exec -it xrd-single-xrd-vrouter-0 -- xr

User Access Verification

Username: cisco
Password: 


RP/0/RP0/CPU0:xrd1#sh ip int brie
Thu Jun 15 00:19:21.901 UTC

Interface                      IP-Address      Status          Protocol Vrf-Name
TenGigE0/0/0/0                 unassigned      Shutdown        Down     default 
RP/0/RP0/CPU0:xrd1#
```
