### Shared Conda Environments and Package Management

This section describes how to set up **shared Conda environments** on the `/fss` filesystem so that all nodes in the cluster (login, controller, compute, monitoring) can use the same software stacks. It also briefly introduces **ODSC** (Oracle Data Science CLI) as a future option for more automated environment management.

---
### 1. Create a shared directory on `/fss`

First, create a user-specific shared directory on the NFS-mounted filesystem `/fss` that is accessible across all nodes.

Run the following as `root` on the login node:

```bash
root@super-ant-login:~# mkdir -p /fss/kaushik
root@super-ant-login:~# sudo chown kaushik:privilege /fss/kaushik
root@super-ant-login:~# sudo chmod 755 /fss/kaushik
```

This creates `/fss/kaushik` as a shared, user-owned directory where we will install Miniconda and environment prefixes that can be activated from any node.

---
### 2. Install Miniconda under `/fss/kaushik`

Install Miniconda into a shared prefix so that all nodes can see the same Conda installation.

Run the following as user `kaushik`:

```bash
# As kaushik
mkdir -p /fss/kaushik/envs

wget -q -P /tmp https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

bash /tmp/Miniconda3-latest-Linux-x86_64.sh \
  -b \
  -p /fss/kaushik/miniconda3

rm /tmp/Miniconda3-latest-Linux-x86_64.sh

# Load conda into your shell
source /fss/kaushik/miniconda3/etc/profile.d/conda.sh

# (Optional but recommended) tell conda to put envs under /fss/kaushik/envs by default
conda config --add envs_dirs /fss/kaushik/envs
```

Because `/fss` is shared across all nodes, this Miniconda installation and any environments created under `/fss/kaushik/envs` will be available everywhere (as long as `/fss` is mounted and `conda.sh` is sourced).

---
### 3. Oracle ADS environment under `/fss/kaushik/envs/oracle-ads`

Instead of creating a named environment under `$HOME`, we create a **prefix environment** under `/fss`. This makes it easy to share the environment across nodes and avoid home-directory–specific paths.

```bash
# Make sure conda is loaded
source /fss/kaushik/miniconda3/etc/profile.d/conda.sh

# Create env at /fss/kaushik/envs/oracle-ads
conda create -y -p /fss/kaushik/envs/oracle-ads python==3.10

# Activate by path
conda activate /fss/kaushik/envs/oracle-ads

# Install ADS with opctl extras
python3 -m pip install "oracle-ads[opctl]"

# Sanity check
ads opctl -h
```

Later, you can activate this environment from **any node** (with `/fss` mounted) using:

```bash
source /fss/kaushik/miniconda3/etc/profile.d/conda.sh
conda activate /fss/kaushik/envs/oracle-ads
```

This gives you a consistent `oracle-ads` environment cluster-wide, suitable for running OCI Data Science / ADS workflows and `ads opctl` commands from multiple nodes.

---
### 4. PyTorch + CUDA 12.1 environment under `/fss/kaushik/envs/pytorch_cuda121`

Similarly, we create a shared PyTorch environment with CUDA 12.1 support under `/fss`.

```bash
# Load conda
source /fss/kaushik/miniconda3/etc/profile.d/conda.sh

# Create env in /fss
conda create -y -p /fss/kaushik/envs/pytorch_cuda121 python=3.10

# Activate
conda activate /fss/kaushik/envs/pytorch_cuda121

# Install PyTorch + CUDA 12.1
conda install -y \
  pytorch torchvision torchaudio pytorch-cuda=12.1 \
  -c pytorch -c nvidia
```

Later, from any node:

```bash
source /fss/kaushik/miniconda3/etc/profile.d/conda.sh
conda activate /fss/kaushik/envs/pytorch_cuda121

# Inspect available envs and verify paths
conda info --envs
ls -d /fss/kaushik/envs/*
```

This environment layout ensures:

- All nodes use the same PyTorch/CUDA stack.
- Environments live under `/fss` and are not tied to a particular node’s local disk.
- You can easily copy or snapshot `/fss/kaushik/envs` for backup or migration.

---
### 5. ODSC (Oracle Data Science CLI) — for future exploration

For longer-term, production-style workflows, OCI provides a more integrated way to manage environments and jobs using **ODSC** (Oracle Data Science CLI).

ODSC is a **user-facing tool**. Users interact with commands such as:

- `odsc conda publish`
- `odsc job run`
- `odsc model deploy`

ODSC wraps standard Conda functionality but adds critical **OCI-specific logic** for:

- Publishing and versioning Conda environments in **OCI Object Storage**.
- Reusing environments across jobs, notebooks, and models.
- Managing environment creation/copying within your OCI tenancy when new compute nodes are provisioned.
- Ensuring that HPC or data science jobs scale out using a consistent, pre-published environment.

In practice, ODSC acts as a **configuration + packaging tool** that tells the OCI platform *which environment* to use when it automatically provisions a new compute node to run your code at scale.

In an HPC context, you could potentially integrate ODSC with the `oci-hpc` stack by adding a role under the Ansible playbooks so that the ODSC tooling is available on all nodes:

> Example path in the quickstart repo:  
> <https://github.com/oracle-quickstart/oci-hpc/tree/master/playbooks/roles>

The following screenshots show example ODSC / OCI Data Science environment and job configuration screens:

<img src="images/3_package/1.png" alt="Example ODSC or OCI Data Science environment management screen" width="75%">

<img src="images/3_package/2.png" alt="Example ODSC or OCI Data Science job or model configuration screen" width="75%">

---
### 6. References and further reading

For future reference and deeper integration:

- ODSC and Conda environment concepts in OCI Data Science:  
  <https://docs.oracle.com/en-us/iaas/Content/data-science/using/conda_understand_environments.htm>
- Accelerated Data Science (ADS) documentation:  
  <https://accelerated-data-science.readthedocs.io/en/latest/>
- `opctl` CLI reference (for ADS/ODSC jobs and pipelines):  
  <https://docs.oracle.com/en-us/iaas/tools/oci-cli/3.71.0/oci_cli_docs/cmdref/opctl.html>
- Publishing modules in OCI Object Storage for team reuse and sample workflows:  
  <https://github.com/oracle-samples/oci-data-science-ai-samples>

At this point, you have:

- A shared Miniconda installation on `/fss`.  
- Two shared environments for **Oracle ADS** and **PyTorch + CUDA 12.1**.  
- Pointers to **ODSC** as a more scalable, OCI-native option for managing and deploying environments across nodes.
