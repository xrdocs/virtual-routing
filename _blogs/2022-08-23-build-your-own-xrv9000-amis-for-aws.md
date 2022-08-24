---
published: true
date: '2022-08-23 05:43 +0530'
title: Build your own XRv9000 AMIs for AWS
author: Akshat Sharma
excerpt: >-
  If you have access to an XRv9000 iso and are looking to deploy it on AWS -
  then this blog will walk you through the steps needed to build the AWS AMI
  yourself.
tags:
  - iosxr
  - linux
  - xrv9k
  - xrv9000
  - AWS
  - public cloud
  - virtual
  - AMI
  - image
  - terraform
  - Ansible
position: hidden
---

{% include base_path %}
{% include toc %}

# Introduction

Cisco XRv9000 is available as a public AMI in the AWS marketplace today. More details here:

>[XRv9000 7.3.3 AMI in AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-ygifeqmzmkqja)  

For most exploratory or production use-cases, this is the ideal AMI to utilize and get hands-on with XRv9000 on AWS.
However, there may be several situations when it might be required to build your own AMI from an available XRv9000 ISO (XRv9k release not yet in the market, private engineering builds meant for future release etc.) This is possible with the help of the automated xrv9k-amibuilder published to Github here:  

> [https://github.com/akshshar/xrv9k-amibuilder](https://github.com/akshshar/xrv9k-amibuilder)  


The AMI builder can be run from your laptop or any client machine with minimal dependencies - only docker must be installed. The actual build takes place on AWS - a working AWS account with Access+Secret key for automated workflows is therefore required.  

**Important**: The AMIs built using this tool (and not obtained through the Marketplace) do not come with Cisco support.
For any support expectations for XRv9000 on AWS, always use official AMIs released in the AWS Marketplace: [https://aws.amazon.com/marketplace/pp/prodview-ygifeqmzmkqja](https://aws.amazon.com/marketplace/pp/prodview-ygifeqmzmkqja).
{: .notice-warning}
  
  
# Building an XRv9000 AMI

Build a Cisco XRv9000 AMI image from an xrv9k ISO to be able to spin up on AWS EC2.
The build code utilizes AWS S3 and temporary instances on AWS in tandem to generate the AMI from the ISO.

**Note**: You need a working AWS account to utilize the build code. While the code launches instances on AWS temporarily and also tries to minimize cost by utilizing spot instances where applicable - the user should be aware that each build flow may incur a minor cost, depending on the type of AWS contract the user has.


## Requirements: Setting up the Client Machine (Laptop/Server)

### Set up Routing to AWS

The client machine may be your laptop or any server/machine thas has internet access capable of reaching AWS public ip addresses.
You can view the block of public IP addresses that AWS uses by navigating here:  <https://ip-ranges.amazonaws.com/ip-ranges.json> and setting up routing appropriately.


### Compatible OS

Technically the entire build runs using docker containers and nothing needs to be installed on the client machine except for docker itself. Since docker is supported on Linux, MacOSX and Windows, you should be able to run the build code on any of these operating systems once docker is up and running

### Install Docker Engine

Navigate here: <https://docs.docker.com/engine/install/> to install Docker Engine for the OS running on your selected Client Machine. The instructions below capture the build flow from MacOSX.


### Fetch your AWS Credentials
You will need to set up your AWS credentials before running the code. So fetch your Access Key and Secret Key as described here for programmatic access to AWS APIs:
<https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys>  
and keep them ready.

### Download the xrv9000 ISO to the client machine

Typically the xrv9000 ISOs are available for software download on cisco.com here: 
<https://software.cisco.com/download/home/286288939/type/280805694/release/>

**Note**:  Not all software releases of xrv9000 on cisco.com are guaranteed to work on AWS even if converted properly. Usually, releases go through their own testing process for AWS specific deployment and the tested releases are released on AWS marketplace.  A list of working releases are specified at the end of this README.md.  
{: .notice--info}



## Working with xrv9k-amibuilder

### Clone the git repo

```bash
aks::~$git clone https://github.com/akshshar/xrv9k-amibuilder
Cloning into 'xrv9k-amibuilder'...
remote: Enumerating objects: 18, done.
remote: Counting objects: 100% (18/18), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 18 (delta 1), reused 18 (delta 1), pack-reused 0
Unpacking objects: 100% (18/18), done.
aks::~$
aks::~$
aks::~$cd xrv9k-amibuilder/
aks::~/xrv9k-amibuilder$
aks::~/xrv9k-amibuilder$tree .
.
├── README.md
├── amibuilder
├── ansible
│   ├── build_ami.yml
│   ├── build_qcow2.yml
│   ├── scripts
│   │   ├── install_qcow2.sh
│   │   └── sunstone.sh
│   └── ssh_config
├── ansible.cfg
├── aws
│   └── credentials
├── iso
│   └── PlaceISOHere.md
├── main.tf
├── scripts
│   ├── install_qcow2.sh
│   └── sunstone.sh
├── ssh
│   └── PlaceSSHKeysHere.md
└── variables.tf

6 directories, 16 files
aks::~/xrv9k-amibuilder$
```


### Copy xrv9000 ISO to iso/ in the git repo

Copy the xrv9000 ISO that you intend to convert into an AMI to the `iso/` directory of the cloned git repo:

For example, I fetched the CCO image from cisco.com for the 6.6.2 release here:   

```bash
aks::~/Downloads$ls -l xrv9k-fullk9-x.vrr-6.6.2.iso
-rwxr-x---@ 1 akshshar  staff  1117378560 Apr 27  2019 xrv9k-fullk9-x.vrr-6.6.2.iso
aks::~/Downloads$

```  

Copy it to the `iso/` directory:  


```bash
aks::~/Downloads$ls -l xrv9k-fullk9-x.vrr-6.6.2.iso
-rwxr-x---@ 1 akshshar  staff  1117378560 Apr 27  2019 xrv9k-fullk9-x.vrr-6.6.2.iso
aks::~/Downloads$
aks::~/Downloads$
aks::~/Downloads$cp xrv9k-fullk9-x.vrr-6.6.2.iso ~/xrv9k-amibuilder/iso/
aks::~/Downloads$
aks::~/Downloads$
aks::~/Downloads$ls -l  ~/xrv9k-amibuilder/iso/
total 2182392
-rw-r--r--  1 akshshar  staff          76 Apr 30 03:11 PlaceISOHere.md
-rwxr-x---@ 1 akshshar  staff  1117378560 Apr 30 03:17 xrv9k-fullk9-x.vrr-6.6.2.iso
aks::~/Downloads$
aks::~/xrv9k-amibuilder$tree .
.
├── README.md
├── amibuilder
├── ansible
│   ├── build_ami.yml
│   ├── build_qcow2.yml
│   ├── scripts
│   │   ├── install_qcow2.sh
│   │   └── sunstone.sh
│   └── ssh_config
├── ansible.cfg
├── aws
│   └── credentials
├── iso
│   ├── PlaceISOHere.md
│   └── xrv9k-fullk9-x.vrr-6.6.2.iso
├── main.tf
├── scripts
│   ├── install_qcow2.sh
│   └── sunstone.sh
├── ssh
│   └── PlaceSSHKeysHere.md
└── variables.tf

6 directories, 16 files
aks::~/xrv9k-amibuilder$

```


### Set up the AWS credentials

As explained in the requirements section, once you have the Access Key and Secret Key associated with your account ready, fill out `aws/credentials` file in the git repo:

```bash
aks::~/xrv9k-amibuilder$  cat aws/credentials 
[default]
aws_access_key_id =
aws_secret_access_key =

```

### Copy rsa key pair of your Client Machine to ssh/

Generate an RSA key pair:
private-key filename:  id_rsa 
public-key filename: id_rsa.pub

Follow the instructions relevant to the OS of your client machine to do so.
Then copy the key files over to the `ssh/` directory of the cloned git repo:

```bash
aks::~/xrv9k-amibuilder$cp ~/.ssh/id_rsa* ssh/
aks::~/xrv9k-amibuilder$
aks::~/xrv9k-amibuilder$tree ./ssh
./ssh
├── PlaceSSHKeysHere.md
├── id_rsa
└── id_rsa.pub

0 directories, 3 files
aks::~/xrv9k-amibuilder$
```

### Customize variables.tf to match your setup

The  `variables.tf` file is used by Terraform to spin up the build infrastructure on AWS in the right region + availability-zone and also to name the generated artifacts appropriately. See the file dump below.

Make sure `xr_version` and `xrv9k_iso_name` are properly updated to match the version used for the AMI image and to match the copied ISO file name in the `iso/` directory respectively.

Also, based on the OS you use for the client machine, you may need to modify the `ssh_key_public` and `ssh_key_private` variables to correctly reflect the path to the key pair files you copied to the `ssh/` directory earlier.

**Note**: All regions and availability zones of AWS do not support all types of instances. As noted in the variables file, we use two types of temporary instances during the build process: `m5zn.metal` and `m4.large`. These are currently supported in the us-west-2 region and us-west-2a availability zone as specified in the default variables.tf file. If you intend to modify the region/az settings, make sure the required instance types are supported there.  
{: .notice--info}.   


```bash
aks::~/xrv9k-amibuilder$cat variables.tf 
variable "xr_version" {
  default = "662"
}

variable "xrv9k_iso_name" {
  default = "xrv9k-fullk9-x.vrr-6.6.2.iso"
}

variable "ssh_key_public" {
  default     = "./ssh/id_rsa.pub"
  description = "Path to the SSH public key for accessing cloud instances. Used for creating AWS keypair."
}

variable "ssh_key_private" {
  default     = "./ssh/id_rsa"
  description = "Path to the SSH public key for accessing cloud instances. Used for creating AWS keypair."
}

variable "aws_region" {
  default = "us-west-2"
}

variable "aws_az" {
  default = "us-west-2a"
}

variable "ami_builder_instance_type" {
  default = "m5zn.metal"
}

variable "ami_snapshot_instance_type" {
  default = "m4.large"
}

variable "aws_ami_ubuntu1604" {
  type = map(string)

  default = {
    "us-west-2"      = "ami-c62eaabe"
  }
}

variable "s3_iso_bucket" {
    default = "xrv9k-iso-bucket"
}

```

## Enabling Debugs
The logs thrown on stdout by Terraform are usually good enough to see what's happening with the build process.
But if you'd like to enable verbose debugging then, set the DEBUG environment variable to a non-zero value before initiating the build:

```bash
export DEBUG=1

```
## Disabling Debugs

Set DEBUG to 0 to disable debugs:

```bash
export DEBUG=0

```

## Building the AMI

Once you have taken care of the requirements and steps listed below, run the builder code using the `amibuilder` executable, as shown below:

```bash
aks::~/xrv9k-amibuilder$./amibuilder 
####################################################
Initializing the build process ...
####################################################

Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v3.37.0...
- Installed hashicorp/aws v3.37.0 (self-signed, key ID 34365D9472D7468F)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
####################################################
Starting build ... 
####################################################

aws_key_pair.aws_keypair: Creating...
aws_s3_bucket.isobucket: Creating...
aws_default_vpc.default: Creating...
aws_key_pair.aws_keypair: Creation complete after 2s [id=xrv9k_aws_amibuilder_20210429223814]
aws_s3_bucket.isobucket: Still creating... [10s elapsed]
aws_default_vpc.default: Still creating... [10s elapsed]
aws_s3_bucket.isobucket: Still creating... [20s elapsed]
aws_default_vpc.default: Still creating... [20s elapsed]
aws_s3_bucket.isobucket: Creation complete after 24s [id=xrv9k-iso-bucket-20210429223814]
aws_default_vpc.default: Creation complete after 25s [id=vpc-4f2c5128]
aws_security_group.server_sg: Creating...

```




## Supported XR Releases

Here "Supported" implies, XR release images obtained from cisco.com that can run unhindered on AWS by creating an AMI using this tool and the build process has been tested.

xrv9000 IOS-XR releases supported by xrv9k-amibuilder:

  * 6.3.1, 6.3.2
  * 6.4.2 
  * 6.5.2
  * 6.6.2
  * 7.3.2  
  * 7.3.3

The above list will grow as soon as successful builds are verified.
