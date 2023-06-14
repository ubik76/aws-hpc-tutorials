+++
title = "Set up AWS ParallelCluster foundation on HPC6a"
date = 2022-04-10T10:46:30-04:00
weight = 35
tags = ["tutorial", "ParallelCluster", "initialize"]
+++

Typically, to configure AWS ParallelCluster, you use the interactive command **[pcluster configure](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-configuring.html)** to provide the information, such as the AWS Region, VPC, Subnet, and [Amazon EC2](https://aws.amazon.com/ec2/) Instance Type.
For this workshop, you will create a custom configuration file to include the HPC specific options for this lab.

In this section, you will set up the foundation (for example network, scheduler, ...) required to build the ParallelCluster config file in the next section.

Don't skip these steps, it is important to follow each step sequentially, copy paste and run these commands

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
export REGION=us-east-2
export availability_zone_id=use2-az2
export VPC_ID=$(aws ec2 --region ${REGION} describe-vpcs --filter Name=isDefault,Values=true --output json | jq -r .Vpcs[0].VpcId)
export SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=availabilityZoneId,Values=$availability_zone_id" --region=$REGION --output json | jq -r .Subnets[0].SubnetId)
```

#### Build the Cluster configuration file

Next, you build a configuration to generate an optimized cluster to run typical “tightly coupled” HPC applications.

```bash
cat > my-cluster-config.yaml << EOF
HeadNode:
  InstanceType: c6a.4xlarge
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
            - InstanceType: hpc6a.48xlarge
          MinCount: 0
          MaxCount: 2
          DisableSimultaneousMultithreading: true
      Networking:
        SubnetIds:
          - ${SUBNET_ID}
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
      DeploymentType: PERSISTENT_2
      PerUnitStorageThroughput: 1000
EOF
```

