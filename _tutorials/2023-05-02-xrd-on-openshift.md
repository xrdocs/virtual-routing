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

## TuneD

## Performance Profile

# Deploy XRd Control Plane
