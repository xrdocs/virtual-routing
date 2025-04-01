---
published: true
date: '2025-03-27 16:43 -0700'
title: XRd Control Plane on Openshift
author: Lawrence Troup
excerpt: Overview of running XRd Control Plane on Red Hat OpenShift
position: hidden
---
{% include base_path %}
{% include toc %}

# Introduction

This pages contain resources to guide you through running XRd Control Plane in Red Hat OpenShift. Starting from an operational OpenShift cluster, the instructions explain how to configure machines in the cluster to be suitable for running the XRd Control Plane workload and how to deploy an XRd Control Plane instance.

OpenShift is Red Hat's Kubernetes offering.

The OpenShift documentation should be referred to alongside this guide for more information.

These instructions walk through how to configure worker nodes in an OpenShift cluster for running XRd, and how to deploy XRd on those nodes.

# Content

* [XRd Control Plane requirements](#xrd-control-plane-requirements)
* [Configuring workers in an OpenShift cluster for running XRd Control Plane](#configuring-a-node-for-xrd-control-plane)
* [Creating SR-IOV networking resources](#sr-iov-network-resources)
* [Creating a Namespace and Service Account](#namespace-and-service-account)
* [Running XRd Control Plane](#running-xrd-control-plane)

The values used in this document when deploying XRd Control Plane, such as number of CPUs and memory allocation, are representative of possible requirements for production workloads. However, for an actual production deployment these values should be tuned to the requirements, such as for scale, of the deployment.
{: .notice--warning}

## Resources

In addition to the documentation here, other key resources for bringing up XRd deployments are:

* The [XRd Helm Charts](https://github.com/ios-xr/xrd-helm), also available in a [Helm repository](https://ios-xr.github.io/xrd-helm), should be used to run XRd in any Kubernetes cluster.

## Prerequisites

These instructions assume a pre-existing OpenShift setup that meets the following requirements:

* There is an operational and reachable OpenShift cluster
* The cluster has all the OpenShift operators, such as the Node Tuning Operator and Cluster Networking Operator, installed that are installed by default with an OpenShift installation
* The cluster contains at least one worker node suitable for running XRd. That is, there must be a worker node that is able to meet the other requirements listed in [XRd Control Plane requirements](#xrd-control-plane-requirements)
* IP addresses on the cluster's internal network are reachable, for example using a Kubernetes Service (see [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/service/)) or allowing direct SSH access to worker nodes in the cluster
* A PersistentVolume must be set up and usable by Pods on the worker node(s) to be used by XRd (see [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/))

These instructions use the OpenShift CLI (`oc`) (see [OpenShift documentation](https://docs.openshift.com/container-platform/4.14/cli_reference/openshift_cli/getting-started-cli.html)) to interact with the cluster.

In order to configure worker nodes for running XRd, some information about the worker machine is required. These instructions use the `oc debug` command to gather this information, but other methods, such as direct ssh access to the worker machine, would also work.

These instructions are for OpenShift 4.14 and 4.16, but more recent OpenShift versions are expected to be similar.
{: .notice--info}

## Running commands from the instructions

Throughout the instructions several bits of output from OpenShift commands should be noted down for use in future commands. Values in angle brackets, e.g. `<node name>`, that are present in code blocks should be substituted with values taken from earlier command output, or with other values as specified.

# XRd Control Plane requirements

XRd Control Plane places hardware requirements on the host. The requirements are summarized in this section to give context for the OpenShift configuration done in subsequent sections.

The requirements of XRd Control Plane fall into the following categories:

* [Host kernel requirements](#host-kernel-requirements)
* [CPUs](#cpus)
* [Memory](#memory)
* [Disk space](#disk-space)
* [Core file handling](#core-file-handling)

## Host kernel requirements

XRd Control Plane has various required host kernel parameters (i.e. `sysctl` settings) that must be set on the host machine.

## CPUs

XRd Control Plane requires that the host has an x86_64 CPU with at least two CPU cores. Furthermore, each XRd Control Plane instance running on the host requires at least one CPU core.

In this guide, 4 CPU cores are requested with a limit of 8. In deployments, the CPU allocation should vary according to BGP scale.

## Memory

XRd Control Plane requires that the host has at least 4GB of memory. Furthermore, each XRd Control Plane instance running on the host requires at least 2GB of memory. (Higher scale production deployments will require more memory).

In this guide, 8GB of memory is requested with a limit of 12GB.

## Disk space

Each XRd instance requires at least 3GB of disk space. This can either be in a persistent volume or ephemeral. These instructions assume that a persistent volume with read-write access and sufficient capacity is available on the worker nodes where XRd is to be deployed. If deploying more than one XRd Control Plane, either a persistent volume with "many" access or a dedicated persistent volume per XRd is required.

## Core file handling

When running XRd, the host machine must have a robust core handling system in place to avoid disk exhaustion and availability issues. One of the considerations of this strategy is how much disk space is available on each worker node. For XRd Control Plane, the worker node must have at least three times the maximum memory allocation plus disk size of all deployed XRd Control Planes. I.e.

```
total_disk_size = <disk-space-for-host> +
                   <disk-space-for-xrd-instances> +
                   <disk-space-for-core-files>
                = <disk-space-for-host> +
                   (<number-of-xrd-instances> * <per-instance-disk-space>) +
                   (3 * <number-of-xrd-instances> * <per-instance-ram>)
```

So for example, if:

* There are 2 XRd Control Plane instance
* Each instance has 4GB disk space and 6GB RAM
* The worker node host requires 5GB disk space

Then the total disk size required for the worker node can be calculated as:

```
total_disk_size = 5GB + (2 * 4GB) + (3 * 2 * 6GB) = 49GB
```

In these instructions, per-worker-node core file handling is set up using `systemd`. Users should consider what core file handling strategy suites their needs in multi-node deployments (see [this Red Hat blog](https://www.redhat.com/en/blog/a-guide-to-core-dump-handling-in-openshift)).

# Configuring a node for XRd Control Plane

XRd needs some of the default values used by OpenShift to be tuned (both machine configuration and deployment options) to meet the requirements listed in [XRd Control Plane requirements](#xrd-control-plane-requirements). This section details the additional configuration required to tune worker nodes in an OpenShift cluster for use with XRd.

All these configurations are compatible with standard OpenShift deployments. They are not expected to conflict with the requirements of any other (non-XRd) Pods, so should not prevent deployment of other Pods on the same worker node.

To meet XRd's requirements, two configurations must be applied to the worker nodes to modify kernel parameters, as detailed in following subsections:

* [Machine Config](#machine-config)
* [Tuned](#tuned)

In these instructions, the node configuration is applied to all worker nodes in the cluster. Custom Machine Config Pools (see [OpenShift documentation](https://docs.openshift.com/container-platform/4.14/post_installation_configuration/machine-configuration-tasks.html#understanding-the-machine-config-operator) for more information) can be used to target the configuration if this behavior is not desired.

## Machine Config

With the information gathered, the first step is to create a Machine Config that sets the number of inotify max user watches and instances.

Create a file `machine_config.yaml` containing the following:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-master-<config name>
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
        - path: /etc/sysctl.d/inotify.conf
          contents:
            source: data:,fs.inotify.max_user_watches%20%3D%2065536%0Afs.inotify.max_user_instances%20%3D%2065536%0A
          mode: 420
          overwrite: true
```

where `<config name>` is a unique name that shall be used for all the configuration of the worker nodes. This sets both `fs.inotify.max_user_watches` and `fs.inotify.max_user_instances` to 65536. Note that the requirement is 4000 per XRd Control Plane instance.

Then apply this config with (the node will restart):

```bash
oc apply -f machine_config.yaml
```

When the Machine Config is applied, the following command will show that the Machine Config Pool is updating:

```bash
$ oc get machineconfigpool worker
NAME     CONFIG                            UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
worker   rendered-worker-(random string)   False     True       False      1              0                   0                     0                      34d
```

Once the Machine Config Pool has finished updating, the same command will show that the config is `UPDATED`.

## TuneD

A TuneD profile needs to be applied to set host kernel parameters to meet the requirements discussed in [Host kernel requirements](#host-kernel-requirements).

Create a file `tuned.yaml` containing the following:

```yaml
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: <config name>
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - data: |
      [main]
      summary=Configuration to set sysctl settings
      [sysctl]
      kernel.randomize_va_space=2
      net.core.rmem_max=67108864
      net.core.wmem_max=67108864
      net.core.rmem_default=67108864
      net.core.wmem_default=67108864
      net.core.netdev_max_backlog=300000
      net.core.optmem_max=67108864
      net.ipv4.udp_mem="1124736 10000000 67108864"
      kernel.core_pattern="|/lib/systemd/systemd-coredump %P %u %g %s %t 9223372036854775808 %h"
    name: <config name>
  recommend:
  - machineConfigLabels:
      machineconfiguration.openshift.io/role: "worker"
    priority: 19
    profile: <config name>
```

The `kernel.core_pattern` argument sets up core file handling using `systemd` as described in the [requirements section](#core-file-handling).

Apply this TuneD profile with:

```bash
oc apply -f tuned.yaml
```

Verify that the TuneD profile has been applied successfully with:

```bash
$ oc get profile <node name> -n openshift-cluster-node-tuning-operator
NAME           TUNED           APPLIED   DEGRADED   AGE
(node name)    (config name)   True      False      63m
```

where `<node name>` is the name of a worker node in the cluster (the list of all available nodes in the cluster is given by the command `oc get nodes`). This should be checked for all worker nodes. The profile should be successfully applied (i.e. `DEGRADED` should be false). The worker nodes are now setup so that XRd can be deployed on it. However, before we do so we shall create SR-IOV network resources to use for networking.

# SR-IOV network resources

In OpenShift deployments, XRd Control Plane is able to use SR-IOV virtual functions (VFs) for its data and management interfaces. To use VFs in OpenShift, SR-IOV networking resource pools must first be created. This section runs through the creation of SR-IOV network resource pools to be used by XRd Control Plane.

The following example uses the OpenShift SR-IOV Network Operator. Instructions for installing this operator can be found [here](https://docs.openshift.com/container-platform/4.14/networking/hardware_networks/installing-sriov-operator.html#installing-sriov-operator).

In this example, we will create a resource pool of four VFs on a single physical interface.

We will create resource pools on a per-node basis. First, we must gather information about the available interfaces.

## Gathering information

The `oc debug` command is used to gather interface information. Enter the debug pod for the worker node with

```bash
oc debug node/<node name>
```

To get a list of physical networking devices available on the worker node, run

```bash
ls -l /sys/class/net/ | grep -v virtual | grep -oE "net/(.*)" | sed -e 's_net/__'
```

This returns a list of the Linux interface names of the physical networking devices, make a note of these. To find out the device-type and PCI address that an interface with name `<interface name>` has, run

```bash
ls /sys/bus/pci/devices/*/net | grep <interface name> -B 1 | grep -oE '([0-9a-fA-F]{2}\:[0-9a-fA-F]{2}\.[0-9])' | xargs lspci -s
```

## Creating SR-IOV resource pools

 Create the file `<network node policy name>.netnodepolicy.yaml`, where `<network node policy name>` is a unique name for the policy. Into this file copy the following:

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: <network node policy name>
  namespace: openshift-sriov-network-operator
spec:
  resourceName: <resource name>
  nodeSelector:
    kubernetes.io/hostname: <node name>
  numVfs: 4
  nicSelector:
    pfNames:
    - <interface name>
  deviceType: netdevice
```

where `<resource name>` is the name of the SR-IOV resource that will be created and `<interface name>` is the Linux name of the physical interface that is to be used from the list of available interfaces found above. Note that more that one interface can be included in the same SR-IOV resource.

To create SR-IOV resources on multiple worker nodes, create SR-IOV Network Node Policies for each node.

When the Sriov Network Node Policy is applied, the operator creates `numVfs` VFs on the available NICs that meet the criteria specified under the `nicSelector` key. Other options for the NIC selector include the device PCI address(es). See [OpenShift documentation](https://docs.openshift.com/container-platform/4.15/networking/hardware_networks/configuring-sriov-device.html#configuring-sriov-device) for more information on options included in the Sriov Network Node Policy.

Apply the Sriov Network Node Policy with

```bash
oc apply -f <network node policy name>.netnodepolicy.yaml
```

Running `oc get node <node name> -o jsonpath='{.status.allocatable}'` will show the number of each resource available. Once the SR-IOV network resources are created, you should see that there are four of `"openshift.io/<resource name>"` available, i.e.

```bash
$ oc get node <node name> -o jsonpath='{.status.allocatable}'
{...,"openshift.io/xrd_sriov_resource":"4",...}
```

# Namespace and Service Account

XRd must run in a custom Namespace and needs to run as a privileged Pod to access some of the host's resources. To achieve this, we will use a Service Account with access to a privileged Security Context Constraint.

This section describes how to set up a Namespace and Service Account for XRd to use.

## Namespace

Kubernetes uses Namespaces to provide logical isolation of resources. The following manifest will be used to create a Namespace called "xrd". Create a file `xrd.namespace.yaml` containing

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: xrd
  labels:
    kubernetes.io/metadata.name: xrd
```

Apply this with:

```bash
oc apply -f xrd.namespace.yaml
```

Once the manifest is applied, verify that the Namespace is created:

```bash
$ oc get project | grep xrd
xrd                               Active
```

Switch to the newly created Namespace (so that future commands use this Namespace by default) by entering:

```bash
$ oc project xrd
Now using project "xrd" on server "(server address)".
```

## Service Account

XRd needs privileged access to some of the host's resources and so needs to be run as a privileged Pod. To do this a Service Account with access to a privileged Security Context Constraint is required. Create a file `xrd.sa.yaml` containing

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: xrd-sa
  namespace: xrd
```

Applying this will create a Service Account named "xrd-sa". Apply it with

```bash
oc apply -f xrd.sa.yaml
```

Verify that the Service Account has been created with

```bash
$ oc get serviceaccount xrd-sa
NAME       SECRETS   AGE
xrd-sa     0         1s
```

Next, we bind the xrd-sa Service Account to a privileged role. To do this, we use the default privileged Security Context Constraint in OpenShift. Create a file `xrd.rolebinding.yaml` containing:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:openshift:scc:privileged
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
subjects:
- kind: ServiceAccount
  name: xrd-sa
  namespace: xrd
```

Apply the manifest with:

```bash
oc apply -f xrd.rolebindings.yaml
```

Verify that xrd-sa is now bound to the privileged role:

```bash
$  oc get clusterrolebinding system:openshift:scc:privileged -o wide
NAME                                                                        ROLE                                                                                    AGE   USERS                                                            GROUPS                                         SERVICEACCOUNTS
system:openshift:scc:privileged                                             ClusterRole/system:openshift:scc:privileged                                             34m                                                                                                                   xrd/xrd-sa
```

Under the `SERVICEACCOUNTS` heading `xrd/xrd-sa` should be listed.

# Running XRd Control Plane

We have now configured the worker nodes and created OpenShift resources required to run XRd. This section describes how to run XRd Pods in this environment.

First, add the XRd Helm repository to the machine from which you are interacting with the cluster with

```bash
helm repo add xrd https://ios-xr.github.io/xrd-helm
```

Verify that the XRd Helm repository has been successfully added with

```bash
$ helm repo list
NAME        	URL
xrd         	https://ios-xr.github.io/xrd-helm
```

The `xrd` repo should be present.

## Installing XRd Control Plane

Create a file `xrd.yaml` containing the following

```yaml
image:
  repository: <repository uri containing Control Plane image>
  tag: <tag>
  pullSecrets:
  - name: <image pull secrets>

config:
  username: <username>
  password: <password>
  # ASCII XR configuration to be applied on XR boot, including configuration for SSH which will be accessible over a Mgmt interface.
  ascii: |
    hostname xrd-1
    ssh server v2

# XRd line interfaces.
interfaces:
- type: sriov
  resource: openshift.io/<resource name>
  config:
    type: sriov
    trust: "on"
    spoofChk: "off"

# Management interfaces. snoopIpv4Address adds detected IP address to XR config to allow SSH access.
mgmtInterfaces:
- type: defaultCni
  chksum: true
  snoopIpv4Address: true

resources:
  requests:
    cpu: '4'
    memory: '8Gi'
  limits:
    cpu: '8'
    memory: '12Gi'

serviceAccountName: xrd-sa

# Persistent storage
persistence:
  enabled: true
  size: 3Gi
  accessModes:
  - ReadWriteOnce
  existingVolume: <persistent volume name>
  storageClass: <storage class name>
```

where:

* `<repository uri containing Control Plane image>` is an image repository containing XRd Control Plane images
* `<tag>` is the image tag to use
* `<image pull secrets>` is a standard Kubernetes Pod imagePullSecrets array ([see here](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/))
* `<username>` a username to log into XR with
* `<password>` a password to log into XR with
* `<persistent volume name>` is the name of the Persistent Volume
* `<storage class name>` is the name of the Storage Class to which the Persistent Volume belongs

The Storage Class and Persistent Volume specified under `persistence` must match the prerequisite Persistent Volume which must have at least 3GB free capacity. In this example, a dedicated (per XRd) Persistent Volume with an access mode of `ReadWriteOnce` is used.

Note that if no persistent storage is given, for example if the `persistence` section is omitted, then ephemeral storage will be used instead. This is not suitable for deployment scenarios due to high-availability considerations.

The `xrd.yaml` Helm values file will be used to install an XRd Control Plane as a Burstable Pod that requests 4 CPU cores and 8GB of memory, with a limit of 8 and 12GB respectively. If desired, XRd Control Plane can be deployed as a Guaranteed Pod by specifying equal requests and limits for the number of CPU cores and amount of memory respectively.

The XRd Control Plane will have a single interface drawn from the `<resource name>` SR-IOV network resource pool created in [SR-IOV network resources](#sr-iov-network-resources) section. Setting `config.trust` to `"on"` for VFs is required to be able receive multicast traffic (and thus is required for multicast based protocols to work such as IPv6 ND). Setting `config.spoofChk` to `"off"` for VFs is required to send packets from other unicast MAC addresses (such as the VRRP vMAC, and thus is required for protocols such as VRRP).

The XRd will also have a management interface that uses the Pod's default veth interface on the cluster network. The applied config allows SSH access to XRd Control Plane at the Pod's IP address.

Multiple interfaces and management interfaces can be requested from multiple SR-IOV resource pools by listing the desired interfaces under the `interfaces` and `mgmtInterfaces` keys, for example:

```yaml
# XRd line interfaces.
interfaces:
- type: sriov
  resource: openshift.io/<resource name>
  config:
    type: sriov
    trust: "on"
    spoofChk: "off"
- type: sriov
  resource: openshift.io/<resource name of another resource pool>
  config:
    type: sriov
    trust: "on"
    spoofChk: "off"

# XRd management interfaces.
mgmtInterfaces:
- type: sriov
  resource: openshift.io/<resource name of another resource pool>
  config:
    type: sriov
    trust: "on"
```

The full range of options supported in the XRd Control Plane Helm values file are documented [here](https://github.com/ios-xr/xrd-helm/blob/main/charts/xrd-control-plane/values.yaml).

Then, install XRd Control Plane with

```bash
helm install xrd xrd/xrd-control-plane -f xrd.yaml
```

## Accessing XRd

The installation steps above start an XRd Pod on a worker node. It will take around a minute to come up as the image is pulled from the repository and XRd boots.

The progress can be monitored manually by looking for the status of the XRd Pod using:

```bash
$ oc get pod xrd-xrd-control-plane-0
NAME                      READY   STATUS    RESTARTS   AGE
xrd-xrd-control-plane-0   1/1     Running   0          4m36s
```

Once the Pod is in `Running` state, we can connect to the Pod using SSH. To do so, first identify the IP address assigned to the Pod by running

```bash
oc get pod xrd-xrd-control-plane-0 --template '{{.status.podIP}}'
```

Make a note of the returned IP address. Then, wait for XR to finish booting. To see the current status, run

```bash
oc logs xrd-xrd-control-plane-0
```

Retry this until the following log is seen:

```bash
$ oc logs xrd-xrd-control-plane-0
(omitted output)
RP/0/RP0/CPU0:Oct 22 16:21:29.194 UTC: ifmgr[229]: %PKT_INFRA-LINK-3-UPDOWN : Interface MgmtEth0/RP0/CPU0/0, changed state to Up
```

Now, the Pod can be accessed via SSH from an end point with access to the cluster-internal network (for example if the network has been exposed via a Kubernetes Service (see [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/service/)) or using the node as a jump host) using

```bash
ssh <username>@<Pod IP>
```

where `<Pod IP>` is the address returned in the previous step. Enter the password `<password>` for access.

Once past the prompt, the status of the XR interfaces can be checked by running `show ip interfaces brief`, which should show a management interface and data interface in the output, similar to:

```bash
RP/0/RP0/CPU0:xrd-1#show ip interfaces brief
Wed Jul 10 12:55:05.810 UTC

Interface                      IP-Address      Status          Protocol Vrf-Name
MgmtEth0/RP0/CPU0/0            (Pod IP)        Up              Up       default
GigabitEthernetE0/0/0/0        unassigned      Shutdown        Down     default
```

The SSH connection can be closed by running `exit`.

# Summary

XRd Control Plane will now be running in Red Hat OpenShift.

This was covered in four steps from a functioning OpenShift cluster:

1. The machine was set up for running XRd using Machine Config and TuneD
2. SR-IOV networking resources were created
3. A namespace and Service Account for running XRd were created
4. An XRd Control Plane workload was deployed on a worker node

# Appendix A: Using physical functions (PFs)

This section replaces the [steps for creating SR-IOV network resources](#sr-iov-network-resources) if physical functions (PFs), instead of VFs, are to be used for network resources. The OpenShift SR-IOV Network Operator cannot be used with PFs. Instead, the SR-IOV Network Device Plugin must be used directly. This section assumes that the SR-IOV Network Device Plugin, the Host-Device CNI, and the Network Resources Injector are all installed on the node.

PF resource bundles are created by configuring the SR-IOV Network Device Plugin using a ConfigMap such as

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: <config name>
  namespace: kube-system
data:
  config.json: |
    {
      "resourceList": [{
          "resourceName": "<resource name>",
          "resourcePrefix": "<resource prefix>",
          "selectors": [{
            "pciAddresses": ["<pci address>"]
          }]
        }
      ]
    }
```

where:

* `<config name>` is a name for the Config Map
* `<resource name>` is a unique name for the resource in the scope of the below resource prefix
* `<resource prefix>` is a prefix for the resource name
  * If none is specified, 'intel.com' is used by default
* `<pci address>` is the PCI address of the PF
  * Alternative resource selectors can be used, see [here](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin?tab=readme-ov-file#configurations)

Note that resource pools of pre-existing VFs can also be created using this method.

Deploying the SR-IOV Network Device Plugin with this ConfigMap then creates the SR-IOV resource bundles. If the deployment is successful, available resources can be be seen via `oc get node <node name> -o json | jq '.status.allocatable'` will show the created SR-IOV resources with the number available:

```bash
$ oc get node <node name> -o json | jq '.status.allocatable'
{
  ...
  "(resource prefix)/(resource name)": "1",
  ...
}
```

When using the SR-IOV Network Device Plugin directly, the desired network `resources` must be requested in the Helm under both the `resources.requests` and `resources.limits` keys, in addition to the interface specification. For example

```yaml
...

resources:
  requests:
    <resource prefix>/<resource name>: 1
    (other requests)
  limits:
    <resource prefix>/<resource name>: 1
    (other limits)

interfaces:
- type: sriov
  resource: <resource prefix>/<resource name>
  config:
    # The SR-IOV CNI cannot be used for PFs, so instead use Host Device CNI
    type: host-device

...
```

in addition to the other config in `xrd.yaml` above. (The requirement to request resources explicitly can be relaxed if the Network Resources Injector is used.)

# Appendix B: Troubleshooting

## OpenShift SR-IOV Network Operator install fails with message `Bundle unpacking failed. Reason: DeadlineExceeded, and Message: Job was active longer than specified deadline`

This is a [known Red Hat bug](https://issues.redhat.com/browse/OCPBUGS-6771).

Workaround:

1. Find the corresponding job and configmap (usually named the same) in the openshift-marketplace and grep for the operator name or keyword in its contents.

```bash
$ oc get job -n openshift-marketplace -o json | jq -r '.items[] | select(.spec.template.spec.containers[].env[].value|contains ("sriov")) | .metadata.name'
```

2. Delete the found job and the corresponding configmap, (named the same as the job) found in the openshift-marketplace namespace.

```bash
$ oc delete job <job-name> -n openshift-marketplace
$ oc delete configmap <job-name> -n openshift-marketplace
```

3. Now delete the subscription

```bash
$ oc delete sub sriov-network-operator-subscription -n openshift-sriov-network-operator
```

Now try and reinstall the subscription.
