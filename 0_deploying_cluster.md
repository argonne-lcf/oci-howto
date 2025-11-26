### Deploying cluster

Deploying a cluster on the Argonne-AI tenant is done using the resource->stack section in OCI

Configurations are provided using the terraform scripts at https://github.com/oracle-quickstart/oci-hpc . Download the master as .zip and upload under terraform configuration.

We basically define the login nodes, controller nodes, monitoring nodes and GPU compute nodes and use all standard configuration from the above oci-hpc terraform or ansible configuration.

Starting the stack in the multinode compartment

<img src="images/0_deploying_cluster/1.png" alt="description" width="40%">

Adding the terraform scripts zip

<img src="images/0_deploying_cluster/2.png" alt="description" width="40%">

Adding the admins laptop ssh keys

<img src="images/0_deploying_cluster/3.png" alt="description" width="40%">

Configuring the login node

<img src="images/0_deploying_cluster/4.png" alt="description" width="40%">

Configuring the monitoring node

<img src="images/0_deploying_cluster/5.png" alt="description" width="40%">

Configuring the filesystem NFS node

All nodes have a common shared file system /config, /home and /fss and varying sizes of local SSDs Note the common shared file system is NFS and not lustre

<img src="images/0_deploying_cluster/6.png" alt="description" width="40%">

Configuring the local SSDs

<img src="images/0_deploying_cluster/7.png" alt="description" width="40%">

Configuring the network
login, controller in public subnet and compute in private subnet
ROCE V2

<img src="images/0_deploying_cluster/8.png" alt="description" width="40%">

Configuring the slurm on the controller node

<img src="images/0_deploying_cluster/9.png" alt="description" width="40%">

health check runs every 5mins and at the start of the job when enabled

<img src="images/0_deploying_cluster/10.png" alt="description" width="40%">

Review, Run Deploy

<img src="images/0_deploying_cluster/11.png" alt="description" width="40%">

Verify the deployed NFS

<img src="images/0_deploying_cluster/12.png" alt="description" width="40%">

Verify the deployed networking at

<img src="images/0_deploying_cluster/13.png" alt="description" width="40%">

Verify the deployed instances

<img src="images/0_deploying_cluster/14.png" alt="description" width="40%">

Copy the IP to login

<img src="images/0_deploying_cluster/15.png" alt="description" width="40%">

Verify the cuda version deployed

<img src="images/0_deploying_cluster/16.png" alt="description" width="40%">

How to terminate the instances. can be rebooted using the mgmt commands on the controller as well.

<img src="images/0_deploying_cluster/17.png" alt="description" width="40%">

important mgmt commands

<img src="images/0_deploying_cluster/18.png" alt="description" width="40%">

Get status on instances - More on monitoring later.

<img src="images/0_deploying_cluster/19.png" alt="description" width="40%">

custom images can be built from https://github.com/oracle-quickstart/oci-hpc-images as well.
