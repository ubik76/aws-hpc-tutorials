+++
title = "Install and run OpenFoam"
date = 2022-04-10T10:46:30-04:00
weight = 60
tags = ["tutorial", "create", "ParallelCluster"]
+++

In this section we will connect to the cluster that you have created in the previous step and then we will install OpenFoam.

### Connect to the HPC cluster

From your AWS Cloud9 terminal, run the following command:

```bash
pcluster ssh -i ../.ssh/pc-intro-key -n hpc-cluster-lab
```

Now that you are connected to the HPC cluster, you can type the SLURM commands, for example
```bash
sinfo
```
will show you the queues that you have configured in the ParallelCluster configuration file.

The **squeue** command will show you the list of jobs that are currently running, or that are waiting in the queue.
```bash
squeue
```
### Install OpenFoam

[OpenFOAM](https://www.openfoam.com/) is the free, open source CFD software developed primarily by OpenCFD Ltd since 2004. It has a large user base across most areas of engineering and science, from both commercial and academic organisations. OpenFOAM has an extensive range of features to solve anything from complex fluid flows involving chemical reactions, turbulence and heat transfer, to acoustics, solid mechanics and electromagnetics.

OpenFOAM is distributed with the GNU GPL v3 Open Source license. You can download and compile OpenFoam on AWS, for exemple using the instructions provided [here](https://github.com/aws-samples/awsome-hpc/blob/main/scripts/install/openfoam_install.sh)

For the scope of this workshop, we will download a precompiled version of OpenFoam.

{{% notice warning %}}
Please note that this version is provided only for the scope of this workshop. Don't use it in a production environment.
{{% /notice %}}

The following commands will create the **apps** directory in our Lustre shared directory and then it will download a tar.gz containing OpenFoam. Then we will download the **motorbike** benchmark and finally we will unzip the tar file.

```bash
mkdir /fsx/apps
cd /fsx/apps

wget https://ee-assets-prod-eu-west-1.s3.eu-west-1.amazonaws.com/modules/216f0fd1f95f4e849947933f8fb1e5ce/v1/openfoam-build.tar.gz

wget https://ee-assets-prod-eu-west-1.s3.eu-west-1.amazonaws.com/modules/216f0fd1f95f4e849947933f8fb1e5ce/v1/motorBikeDemo.tgz

wget https://ee-assets-prod-eu-west-1.s3.eu-west-1.amazonaws.com/modules/216f0fd1f95f4e849947933f8fb1e5ce/v1/motorBikeDemo-72.tgz

tar -xvf openfoam-build.tar.gz
```

### Run your first job

Let's untar the benchmark we have downloaded before:
```bash
cd /fsx/apps
tar -xf motorBikeDemo-72.tgz
cd motorBikeDemo-72
```

You'll see we have a complete OpenFOAM case. The test-case is a modified version of the standard OpenFOAM motorbike tutorial which will create a mesh of approximately 4M cells.

In order to send the job from the head node to the cluster, we need to create a submission script.

Open your preferred text editor and create a script file, name it: **submit.sh**

```bash
vi submit.sh
```
Cut&Paste the content from the section here below:

```bash
#!/bin/bash
#SBATCH --job-name=foam-72
#SBATCH --ntasks=72
#SBATCH --output=%x_%j.out
#SBATCH --partition=queue0
#SBATCH --constraint=c5n.18xlarge

module load openmpi
source /fsx/apps/openfoam/OpenFOAM-v2012/etc/bashrc

cp $FOAM_TUTORIALS/resources/geometry/motorBike.obj.gz constant/triSurface/
surfaceFeatureExtract  > ./log/surfaceFeatureExtract.log 2>&1
blockMesh  > ./log/blockMesh.log 2>&1
decomposePar -decomposeParDict system/decomposeParDict.hierarchical  > ./log/decomposePar.log 2>&1
mpirun -np $SLURM_NTASKS snappyHexMesh -parallel -overwrite -decomposeParDict system/decomposeParDict.hierarchical   > ./log/snappyHexMesh.log 2>&1
mpirun -np $SLURM_NTASKS checkMesh -parallel -allGeometry -constant -allTopology -decomposeParDict system/decomposeParDict.hierarchical > ./log/checkMesh.log 2>&1
mpirun -np $SLURM_NTASKS redistributePar -parallel -overwrite -decomposeParDict system/decomposeParDict.ptscotch > ./log/decomposePar2.log 2>&1
mpirun -np $SLURM_NTASKS renumberMesh -parallel -overwrite -constant -decomposeParDict system/decomposeParDict.ptscotch > ./log/renumberMesh.log 2>&1
mpirun -np $SLURM_NTASKS patchSummary -parallel -decomposeParDict system/decomposeParDict.ptscotch > ./log/patchSummary.log 2>&1
ls -d processor* | xargs -i rm -rf ./{}/0
ls -d processor* | xargs -i cp -r 0.orig ./{}/0
mpirun -np $SLURM_NTASKS potentialFoam -parallel -noFunctionObjects -initialiseUBCs -decomposeParDict system/decomposeParDict.ptscotch > ./log/potentialFoam.log 2>&1s
mpirun -np $SLURM_NTASKS simpleFoam -parallel  -decomposeParDict system/decomposeParDict.ptscotch > ./log/simpleFoam.log 2>&1
```

Now that the script is ready, you can launch your job with the **sbatch** command:

```bash
sbatch submit.sh
```

**Other useful Slurm commands**

+ **squeue** – shows the status of all running jobs in the queue.
+ **sinfo** – shows partition and node information for a system
+ **srun** – run an interactive job
+ **scancel *jobid*** – kill the Slurm job with id=*jobid*

If you type **squeue** you should initially see the following which states that it's in a queue.

```console
[ec2-user@ip-10-0-0-22 motorBikeDemo]$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON) 
                 7   compute  foam-108 ec2-user CF       0:03      3 compute-dy-c5n18xlarge-[1-3]
```

In the back-end this is now sending a signal for EC2 instances to be created (which you can see via your EC2 console). It should take around 4-5 minutes for these to be launched. If you do not see this move to 'R' for running within 5 minutes, check your EC2 console and make sure you have requested an increase to your Service Quotas as described in previous sections.

You should now see:
```console
[ec2-user@ip-10-0-0-22 motorBikeDemo]$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                 7   compute  foam-108 ec2-user R       0:05      3 compute-dy-c5n18xlarge-[1-3]
```

The case is setup to write out the various logs in the 'log' folder and it should take approximately 5 minutes to go through the meshing and solving process.