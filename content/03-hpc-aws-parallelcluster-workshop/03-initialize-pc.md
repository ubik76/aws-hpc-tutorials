+++
title = "c. Set up AWS ParallelCluster foundation"
date = 2022-04-10T10:46:30-04:00
weight = 30
tags = ["tutorial", "ParallelCluster", "initialize"]
+++

Typically, to configure AWS ParallelCluster, you use the interactive command **[pcluster configure](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-configuring.html)** to provide the information, such as the AWS Region, VPC, Subnet, and [Amazon EC2](https://aws.amazon.com/ec2/) Instance Type.
For this workshop, you will create a custom configuration file to include the HPC specific options for this lab.

In this section, you will set up the foundation (for example network, scheduler, ...) required to build the ParallelCluster config file in the next section.

{{% notice info %}}Don't skip these steps, it is important to follow each step sequentially, copy paste and run these commands
{{% /notice %}}

Retrieve network information and set environment variables. In these steps you will also be writing these environment variables into a file in your working directory which can be sourced and set again in case you logout of the session.

Your HPC cluster configuration will need network information such as VPC ID, Subnet ID and create a SSH Key.
To ease the setup, you will use a script for settings of those parameters.
If you have time and are curious, you can examine the different steps of the script.

#### Generate a new key-pair
```bash
aws ec2 create-key-pair --key-name pc-intro-key --query KeyMaterial --output text > ~/.ssh/pc-intro-key
```

```bash
chmod 600 ~/.ssh/pc-intro-key
```

#### Getting your AWS networking information
```bash
export IFACE=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/)
export SUBNET_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${IFACE}/subnet-id)
export VPC_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${IFACE}/vpc-id)
export REGION=$(curl --silent http://169.254.169.254/latest/meta-data/placement/availability-zone | sed 's/[a-z]$//')
```

#### Build the Cluster configuration file

Next, you build a configuration to generate an optimized cluster to run typical “tightly coupled” HPC applications.

```bash
cat > my-cluster-config.yaml << EOF
HeadNode:
  InstanceType: t2.micro
  Networking:
    SubnetId: ${SUBNET_ID}
  Ssh:
    KeyName: pc-intro-key
  LocalStorage:
    RootVolume:
      VolumeType: gp3
      Size: 50
      Encrypted: true
  Iam:
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
  Dcv:
    Enabled: true
Scheduling:
  Scheduler: slurm
  SlurmQueues:
    - Name: queue0
      AllocationStrategy: lowest-price
      ComputeResources:
        - Name: queue0-compute-resource-0
          Instances:
            - InstanceType: c5n.18xlarge
          MinCount: 0
          MaxCount: 2
          DisableSimultaneousMultithreading: true
      Networking:
        SubnetIds:
          - subnet-f6558992
        PlacementGroup:
          Enabled: true
      ComputeSettings:
        LocalStorage:
          RootVolume:
            VolumeType: gp3
  SlurmSettings: {}
Region: $REGION
Image:
  Os: alinux2
SharedStorage:
  - Name: FsxLustre0
    StorageType: FsxLustre
    MountDir: /fsx
    FsxLustreSettings:
      StorageCapacity: 1200
      DeploymentType: SCRATCH_2
EOF
```

