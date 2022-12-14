---
published: true
date: '2022-12-08 15:34 -0800'
title: Deploy XRd on AWS
author: Taran Deshpande
excerpt: Learn how to deploy and interact with XRd running in an EKS cluster
tags:
  - iosxr
  - cisco
  - linux
  - xrd
  - container
  - docker
  - kubernetes
  - AWS
  - virtual
  - EKS
position: top
---
{% include base_path %}
{% include toc %}
## Introduction

With the release of IOS XR 7.8.1, XRd is supported as a Cloud Router and SR-PCE. In this tutorial we will explore how to deploy a topology of XRd cloud routers on Amazon Elastic Kubernetes Service (EKS).
## Prerequisites
We will need the a XRd vRouter image release 7.8.1 or later. The steps to do this was outlined in a previous [tutorial](https://xrdocs.io/virtual-routing/tutorials/2022-08-22-xrd-images-where-can-one-get-them/). In addition, we will need an AWS account and the `aws` [cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured on our local machine. Although these following tools aren't strictly required to launch XRd on EKS, we will use them in our tutorial:

* `kubectl`
* `docker`
* `skopeo`

Clone the [XRd on AWS EKS](https://github.com/ios-xr/xrd-eks) repository for access to cloudformation templates and sample scripts that we will use in this deployment.

```bash
tadeshpa@TADESHPA-M-F92B ~> git clone https://github.com/ios-xr/xrd-eks.git
Cloning into 'xrd-eks'...
remote: Enumerating objects: 87, done.
remote: Counting objects: 100% (87/87), done.
remote: Compressing objects: 100% (61/61), done.
remote: Total 87 (delta 16), reused 86 (delta 15), pack-reused 0
Receiving objects: 100% (87/87), 75.09 KiB | 1.79 MiB/s, done.
Resolving deltas: 100% (16/16), done.
```
From this repository, we will directly use three scripts for the tutorial:
* `publish-ecr` - Places XRd container images in a container registry on AWS
* `publish-s3-bucket` - Uploads example cloudformation templates and AMI assets from the github repository to an S3 bucket on AWS
* `create-stack` - Automates the deployment of an example topology using cloudformation templates

## Publish XRd Images on Elastic Container Registry
EKS needs to pull the XRd image from a container image repository which is accessible to AWS. For this, we will create a repository using AWS's ECR (Elastic Container Registry) service and host the XRd images there.

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main) [1]> ./publish-ecr xrd-vrouter-container-x64.dockerv1.tgz
Login Succeeded!
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-2:655415053484:repository/xrd/xrd-vrouter",
        "registryId": "655415053484",
        "repositoryName": "xrd/xrd-vrouter",
        "repositoryUri": "655415053484.dkr.ecr.us-east-2.amazonaws.com/xrd/xrd-vrouter",
        "createdAt": "2022-12-02T14:06:36-08:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
Getting image source signatures
Copying blob 4952cb498877 done  
Copying config c795cd1683 done  
Writing manifest to image destination
Storing signatures
Image is now available at 655415053484.dkr.ecr.us-east-2.amazonaws.com/xrd/xrd-vrouter:latest
```
Now if we check our ECR Repositories list on the AWS console, we can see that an xrd-vrouter repository has been created
![ECR]({{site.baseurl}}/images/ecr-upload.jpeg)

## Cloudformation Templates and AMI Assets
In the [XRd on AWS EKS](https://github.com/ios-xr/xrd-eks) github repository, a number of Cloudformation templates are provided to assist with deploying XRd on EKS. 

There are three types of templates:

* Atomic Building Block - elements used to build a full deployment stack.
* Composite Building Block - convenient combinations of the atomic building block templates.
* Application - example XRd deployments.
 
The application templates that we deploy create [nested stacks](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html), since they may reference multiple atomic or composite building blocks. Due to this, all necessary templates should be placed in an S3 Bucket so that Cloudformation can easily access them. 

In addition to the cloudformation templates, we will also place the [ami assets](https://github.com/ios-xr/xrd-eks/tree/main/ami_assets/etc), in the same S3 bucket. These AMI Assets are required so an AMI suitable for running XRd vRouter can be created as part of the CloudFormation stack.

## Publishing S3 Bucket
The script `publish-s3-bucket.sh` automates the process of creating the bucket and uploading the relevant resources.

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main) [254]> ./publish-s3-bucket
Creating bucket 655415053484-xrd-quickstart...
{
    "Location": "http://655415053484-xrd-quickstart.s3.amazonaws.com/"
}
Syncing CF templates to bucket...
upload: cf-templates/xrd-ec2-network-interface-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-ec2-network-interface-cf.yaml
upload: cf-templates/xrd-eks-new-vpc-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-eks-new-vpc-cf.yaml
upload: cf-templates/xrd-vpc-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-vpc-cf.yaml
upload: cf-templates/xrd-eks-ami-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-eks-ami-cf.yaml
upload: cf-templates/xrd-overlay-example-existing-eks-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-overlay-example-existing-eks-cf.yaml
upload: cf-templates/xrd-eks-existing-vpc-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-eks-existing-vpc-cf.yaml
upload: cf-templates/xrd-example-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-example-cf.yaml
upload: cf-templates/xrd-eks-node-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-eks-node-cf.yaml
upload: cf-templates/xrd-eks-node-launch-template-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-eks-node-launch-template-cf.yaml
upload: cf-templates/xrd-singleton-existing-eks-cf.yaml to s3://655415053484-xrd-quickstart/xrd-eks/cf-templates/xrd-singleton-existing-eks-cf.yaml
Syncing AMI assets to bucket...
upload: ami_assets/etc/modprobe.d/igb_uio.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/modprobe.d/igb_uio.conf
upload: ami_assets/etc/modprobe.d/vfio.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/modprobe.d/vfio.conf
upload: ami_assets/etc/tuned/xrd-eks-node-variables.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/tuned/xrd-eks-node-variables.conf
upload: ami_assets/etc/tuned/xrd-eks-node/tuned.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/tuned/xrd-eks-node/tuned.conf
upload: ami_assets/etc/modules-load.d/uio.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/modules-load.d/uio.conf
upload: ami_assets/etc/modules-load.d/vfio-pci.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/modules-load.d/vfio-pci.conf
upload: ami_assets/etc/xrd/bootstrap.sh to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/xrd/bootstrap.sh
upload: ami_assets/etc/modules-load.d/igb_uio.conf to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/modules-load.d/igb_uio.conf
upload: ami_assets/etc/tuned/xrd-eks-node/defirqaffinity.py to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/tuned/xrd-eks-node/defirqaffinity.py
upload: ami_assets/etc/xrd/xrd-vrouter-build-ami.sh to s3://655415053484-xrd-quickstart/xrd-eks/ami_assets/etc/xrd/xrd-vrouter-build-ami.sh

Successfully populated S3 bucket
Use the following values in your XRd CloudFormation stack:
  XRd resource S3 bucket name: 655415053484-xrd-quickstart
  XRd resource S3 bucket region: us-east-2
  XRd resource S3 bucket key prefix: xrd-eks/

```
Now when we navigate to the AWS Console and view S3 buckets, we can see that one has been created with the relevant resources uploaded:

![s3-view]({{site.baseurl}}/images/s3-view.jpeg)
![s3-assets]({{site.baseurl}}/images/s3-assets.jpeg)
## Launching the Application
We need an AWS key-pair to access the Bastion and Worker nodes created by stack. If you have not already created one in the region that you are deploying the cluster in, follow this [guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html).
### Launch from AWS Console
Navigate to 'Create Stack' within the AWS console
![create-stack]({{site.baseurl}}/images/create-stack.jpeg)
Next, we will copy the url of the Cloudformation Template that we want to deploy. In this case, we will select `xrd-example-cf.yaml`
![s3-url]({{site.baseurl}}/images/s3-url.jpeg)
#### Create Stack
![step1]({{site.baseurl}}/images/create-stack-step1.jpeg)
#### Specify Stack Details
We will fill in these following parameters:

  * VPC Availability Zones - Choose any two offered AZs.
  * EKS public access endpoint - Chose 'Enabled' to allow external access to the cluster API.
  * EKS remote access CIDR - Choose an IP subnet to allow external access from (or 0.0.0.0/0 to allow access from everywhere).
  * EC2 key pair - Choose the EC2 key pair to install to the bastion and worker nodes (per above).
  * Application - Choose 'Singleton'
  * XR root user name - Specify the root user name for XRd instances.
  * XR root user password - Specify the root user password for XRd instances.
  * XRd resource S3 bucket name - from the `publish-s3-bucket` script output.
  * XRd resource S3 bucket region - as previous.
  * XRd resource S3 bucket key prefix - as previous.
  
![step2]({{site.baseurl}}/images/create-stack-step2.jpeg)
#### Launch Stack
We can skip Step 3 (Configure stack options), and proceed to step 4, where we will acknowledge the capabilities and click submit. 
![step4]({{site.baseurl}}/images/create-stack-step4.jpeg)

The process of launching the stack can take over an hour. We can follow the progress in the events tab of our stack in the AWS Console.
![stack-events]({{site.baseurl}}/images/stack-events.jpeg)

#### Connect to XRd
Navigate to the instances view in the AWS Console.
![xrd-instances]({{site.baseurl}}/images/xrd-instances.jpeg)
We have two instances running - EKSBastion and one worker node. The Bastion host is the point of entry to our network of XRds. The worker node is the host machine of the XRd container.

#### Connect using Kubernetes command line
Update the kubeconfig

```bash
bash-5.2$ aws eks update-kubeconfig --name xrd-cluster --profile xrdocs
Updated context arn:aws:eks:us-east-2:655415053484:cluster/xrd-cluster in /Users/tadeshpa/.kube/config
```
Check the current deployed pods

```bash
bash-5.2$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
xrd-example-xrd-vrouter-0   1/1     Running   0          19m
```
Connect the container using the credentials we supplied during the Cloudformation Stack Launch

```bash
bash-5.2$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
xrd-example-xrd-vrouter-0   1/1     Running   0          19m
bash-5.2$ kubectl exec xrd-example-xrd-vrouter-0 -it -- xr

User Access Verification

Username: cisco
Password: 


RP/0/RP0/CPU0:xrd#show ip interface brief
Thu Dec  8 19:32:22.603 UTC

Interface                      IP-Address      Status          Protocol Vrf-Name
HundredGigE0/0/0/0             10.0.1.11       Up              Up       default 
HundredGigE0/0/0/1             10.0.2.11       Up              Up       default 
HundredGigE0/0/0/2             10.0.3.11       Up              Up       default 
```
### Launch from Command Line
The `create-stack` script will create a more complex application: `xrd-overlay`. This topology includes two routers connected via  an overlay constructed using GRE, IS-IS and L3VPN. Each router is also connected to an alpine linux container workload emulating a NF in an isolated VRF.

![XRd-eks-overlay]({{site.baseurl}}/images/XRd-eks-overlay.jpg)

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main)> ./create-stack -h
Usage: create-stack -u XR_USERNAME -p XR_PASSWORD -k EC2_KEY_PAIR [-a CUSTOM_AMI_ID]

Create an example XRd CF stack
```

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main)> ./create-stack -u cisco -p cisco123 -k xrd-eks
+ aws cloudformation delete-stack --stack-name xrd-Overlay
+ aws cloudformation wait stack-delete-complete --stack-name xrd-Overlay
+ aws cloudformation create-stack --stack-name xrd-Overlay --template-url https://655415053484-xrd-quickstart.s3.us-east-2.amazonaws.com/xrd-eks/cf-templates/xrd-example-cf.yaml --disable-rollback --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND --parameters ParameterKey=XrdS3BucketName,ParameterValue=655415053484-xrd-quickstart ParameterKey=XrdS3BucketRegion,ParameterValue=us-east-2 ParameterKey=XrdS3KeyPrefix,ParameterValue=xrd-eks/ ParameterKey=EKSClusterName,ParameterValue=xrd-cluster 'ParameterKey=AvailabilityZones,ParameterValue="us-east-2a,us-east-2b"' ParameterKey=RemoteAccessCIDR,ParameterValue=0.0.0.0/0 ParameterKey=EKSPublicAccessEndpoint,ParameterValue=Enabled ParameterKey=KeyPairName,ParameterValue=xrd-eks ParameterKey=NodeInstanceType,ParameterValue=m5.2xlarge ParameterKey=Application,ParameterValue=Overlay ParameterKey=Platform,ParameterValue=VRouter ParameterKey=XrdRootUserName,ParameterValue=cisco ParameterKey=XrdRootPassword,ParameterValue=cisco123
{
    "StackId": "arn:aws:cloudformation:us-east-2:655415053484:stack/xrd-Overlay/7f843610-7a65-11ed-90a2-02cf6e66352a"
}
+ aws cloudformation wait stack-create-complete --stack-name xrd-Overlay
+ aws cloudformation list-stacks --query 'StackSummaries[?StackName==`xrd-Overlay` && StackStatus!=`DELETE_COMPLETE`].{Name: StackName, Status: StackStatus, Id: StackId}'
[
    {
        "Name": "xrd-Overlay",
        "Status": "CREATE_COMPLETE",
        "Id": "arn:aws:cloudformation:us-east-2:655415053484:stack/xrd-Overlay/7f843610-7a65-11ed-90a2-02cf6e66352a"
    }
]
+ aws eks update-kubeconfig --name xrd-cluster
Updated context arn:aws:eks:us-east-2:655415053484:cluster/xrd-cluster in /Users/tadeshpa/.kube/config
```

If we navigate to the AWS console, we still see the EKSBastion, but now there are three worker nodes: nodaA, nodeB, and nodeC.

![overlay-xrd]({{site.baseurl}}/images/overlay-xrd.jpeg)

The script will automatically update the kubconfig, so we can see that the four pods are deployed.

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main) [255]> kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
xrd-example-host1-78b766f567-jk5kl   1/1     Running   0          99m
xrd-example-host2-6f948b8c5b-gkdj8   1/1     Running   0          99m
xrd-example-xrd1-0                   1/1     Running   0          99m
xrd-example-xrd2-0                   1/1     Running   0          99m
```

Just like the singleton example, we can use `kubectl` to attach to the containers.

Host1:

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main) [1]> kubectl exec -it xrd-example-host1-78b766f567-jk5kl -- /bin/sh
/ # ip route
default via 169.254.1.1 dev eth0 
10.0.1.0/24 dev net1 scope link  src 10.0.1.10 
10.0.4.0/24 via 10.0.1.11 dev net1 
169.254.1.1 dev eth0 scope link 
```

XRd1:

```bash
tadeshpa@TADESHPA-M-F92B ~/xrd-eks (main)> kubectl exec -it xrd-example-xrd1-0 -- xr

User Access Verification

Username: cisco
Password: 


RP/0/RP0/CPU0:xrd1#show ip interface brief
Tue Dec 13 00:22:49.751 UTC

Interface                      IP-Address      Status          Protocol Vrf-Name
Loopback0                      1.0.0.11        Up              Up       default 
tunnel-ip1                     10.1.2.11       Up              Up       default 
tunnel-ip2                     10.1.3.11       Up              Up       default 
HundredGigE0/0/0/0             10.0.1.11       Up              Up       nfs     
HundredGigE0/0/0/1             10.0.2.11       Up              Up       default 
HundredGigE0/0/0/2             10.0.3.11       Up              Up       default 
RP/0/RP0/CPU0:xrd1#
```

## Deleting the Application

First, navigate to the Stacks page in the AWS Console. Next, we will select the stack that we created in this example, and click delete.

![delete-stack]({{site.baseurl}}/images/delete-stack.png)

It will ask us form confirmation, and then begin to delete all of the AWS resources we created in order to launch the example application.

![confirm-delete-stack]({{site.baseurl}}/images/confirm-delete-stack.png)

There you have it! In this tutorial, we learned how to deploy an XRd topology on EKS. To build an XRd topology specific to your own use case, you can use the *atomic* and *composite* building blocks, to create a new *application* cloudformation template.
{: .notice--success}
