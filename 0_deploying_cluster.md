### Deploying cluster

This document describes how to deploy a GPU-capable compute cluster on the **Argonne-AI** OCI tenant using the **Resource Manager → Stacks** interface in OCI.

The cluster is created using the Terraform and Ansible configurations from the public `oci-hpc` quickstart repository. We define:

- Login nodes  
- Controller node (Slurm controller / management node)  
- Monitoring node  
- Filesystem (NFS) node  
- GPU compute nodes  

and rely on the standard configuration provided by `oci-hpc` as much as possible.

Terraform configuration is provided at:  
https://github.com/oracle-quickstart/oci-hpc

Download the repository (or master branch) as a `.zip` and upload it under **Terraform configuration** when creating the stack.

---
### Starting the stack in the multinode compartment

From the OCI Console, navigate to **Developer Services → Resource Manager → Stacks** and make sure you are in the *multinode* compartment for the Argonne-AI tenant. Create a new stack that will define the HPC cluster.

<img src="images/0_deploying_cluster/1.png" alt="Starting the stack in the multinode compartment" width="75%">

---
### Adding the Terraform scripts zip

When prompted for the configuration source, choose to upload the `.zip` file you downloaded from `oci-hpc`. This zip contains the Terraform modules and Ansible roles used to deploy the cluster.

<img src="images/0_deploying_cluster/2.png" alt="Adding the terraform scripts zip" width="75%">

---
### Adding the admin laptop SSH keys

Provide the public SSH key from the admin’s laptop. This key will be installed on the login node(s) so that you can SSH into the cluster after deployment.

<img src="images/0_deploying_cluster/3.png" alt="Adding the admin laptop SSH keys" width="75%">

---
### Configuring the login node

Configure the login node parameters (shape, image, and any relevant metadata or cloud-init configuration). The login node is the primary entry point for users and administrators.

<img src="images/0_deploying_cluster/4.png" alt="Configuring the login node" width="75%">

---
### Configuring the monitoring node

Configure the monitoring node, which typically runs monitoring and observability tools (e.g., metrics collection, dashboards, alerts). Settings are driven by the `oci-hpc` stack inputs.

<img src="images/0_deploying_cluster/5.png" alt="Configuring the monitoring node" width="75%">

---
### Configuring the filesystem NFS node

Configure the filesystem node that will export NFS shares to all nodes in the cluster.

All nodes have access to a common shared filesystem mounted at:

- `/config`  
- `/home`  
- `/fss`  

In addition, each node is provisioned with local SSDs for node-local scratch storage. **These local SSDs can be configured in different sizes depending on node type** (for example, larger SSDs on GPU compute nodes for datasets and checkpoints, smaller SSDs on login or monitoring nodes).

> Note: The common shared filesystem is **NFS**, not Lustre.

<img src="images/0_deploying_cluster/6.png" alt="Configuring the filesystem NFS node" width="75%">

---
### Configuring the local SSDs

For each node type, configure the local SSDs that will be used as high-performance, node-local scratch. This is often used for temporary datasets, caches, and job-local output that does not need to be shared via NFS.

<img src="images/0_deploying_cluster/7.png" alt="Configuring the local SSDs" width="75%">

---
### Configuring the network

Configure the network layout for the cluster:

- Login and controller nodes reside in a **public subnet** (to allow controlled external access, usually restricted via security rules and SSH).  
- GPU compute nodes reside in a **private subnet**.  
- The backend or **cluster network** is a purpose-built **RDMA over RoCEv2** network that is completely isolated from non-RDMA traffic. This network is used for high-performance MPI, GPU, and storage traffic.

<img src="images/0_deploying_cluster/8.png" alt="Configuring the network" width="75%">

---
### Configuring Slurm on the controller node

Configure Slurm on the controller node. This node usually acts as the Slurm controller and may also perform other management roles (e.g., provisioning, orchestration). Configure partitions, node groups, and other Slurm parameters as exposed by the `oci-hpc` variables.

<img src="images/0_deploying_cluster/9.png" alt="Configuring Slurm on the controller node" width="75%">

---
### Configuring health checks

Enable and configure health checks for the compute nodes. Typically, health checks can be run on a fixed interval and/or at job start.

In this configuration, the health check runs every 5 minutes and at the start of a job when enabled. This helps detect node failures and GPU issues early.

<img src="images/0_deploying_cluster/10.png" alt="Configuring health checks" width="75%">

---
### Review and run deployment

Once all parameters are configured, review the stack configuration and start the deployment. OCI Resource Manager will apply the Terraform plan and create all resources defined for the cluster.

<img src="images/0_deploying_cluster/11.png" alt="Review and run deployment" width="75%">

---
### Verifying the deployed NFS

After the stack reaches an **Active** or **Applied** state, verify that the NFS server is running and that the shared filesystems (`/config`, `/home`, `/fss`) are exported and mounted on the relevant nodes.

<img src="images/0_deploying_cluster/12.png" alt="Verifying the deployed NFS" width="75%">

---
### Verifying the deployed networking

Verify that the virtual cloud network (VCN), subnets, security lists / network security groups (NSGs), and cluster (RDMA) network were created as expected.

<img src="images/0_deploying_cluster/13.png" alt="Verifying the deployed networking" width="75%">

---
### Verifying the deployed instances

Inspect the list of compute instances created by the stack: login node(s), controller node, monitoring node, filesystem node, and GPU compute nodes. All instances should be in a **Running** state.

<img src="images/0_deploying_cluster/14.png" alt="Verifying the deployed instances" width="75%">

---
### Logging in to the cluster

Copy the public IP address of the login node and SSH into it from your laptop using the SSH key you configured earlier:

```bash
ssh -i /path/to/your_private_key ubuntu@<LOGIN_NODE_PUBLIC_IP>
```

<img src="images/0_deploying_cluster/15.png" alt="Copy the IP to login" width="75%">

---
### Verifying the CUDA and GPU software stack

On a GPU compute node, verify that NVIDIA drivers and CUDA are installed correctly. Typical commands:

```bash
nvidia-smi
nvcc --version   # if the CUDA toolkit is installed
```

<img src="images/0_deploying_cluster/16.png" alt="Verify the CUDA version deployed" width="75%">

---
### Terminating or rebooting instances

Instances can be managed directly from the OCI Console, but they can also be rebooted or terminated using the management commands on the controller node, depending on how the `oci-hpc` stack is configured.

Use this when you want to shut down or recycle part of the cluster in a controlled way (e.g., drain nodes from Slurm and then power them off).

<img src="images/0_deploying_cluster/17.png" alt="How to terminate the instances" width="75%">

---
### Important management commands

The controller node usually includes helper scripts and management commands to simplify common operations such as:

- Managing cluster configurations and services  
- Draining and resuming nodes in Slurm  
- Checking health / status of the cluster

<img src="images/0_deploying_cluster/18.png" alt="Important management commands" width="75%">

---
### Getting status on instances

Use the provided management commands or the OCI Console to get detailed status information on all instances. More advanced monitoring (GPU metrics, network performance, etc.) is typically handled by the monitoring node and will be covered in more depth in a separate monitoring document.

<img src="images/0_deploying_cluster/19.png" alt="Get status on instances - more on monitoring later" width="75%">

---
### Custom images

Custom images can be built from:  
https://github.com/oracle-quickstart/oci-hpc-images

These images can be used to bake in site-specific software stacks (HPC libraries, CUDA toolkits, AI frameworks, tools, etc.) and then referenced by the `oci-hpc` Terraform variables when deploying future clusters.
