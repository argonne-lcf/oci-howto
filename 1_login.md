### Logging in to the OCI web portal

Once the PI has added the user to the **Identity** console in OCI, the user will receive an email to activate their account. During activation you must also:

- Download and install the **Oracle Mobile Authenticator** app (on your phone).
- Scan the QR code presented during account activation so that MFA is configured correctly.

<img src="images/1_login/identity_profiles.png" alt="OCI Identity console showing Argonne-AI user profiles" width="75%">

After your account is activated, you can log in to the Argonne-AI tenant at:

- <https://cloud.oracle.com/identity/domains/my-profile?tenant=ArgonneAI&domain=Default&region=us-sanjose-1>

<img src="images/1_login/login.png" alt="OCI login screen for the Argonne-AI tenant" width="25%">

Once you are logged in, you can verify the current instances under the **Argonne-AI** tenant by navigating to **Compute → Instances** in the appropriate compartment.

<img src="images/1_login/instances.png" alt="List of preconfigured instances in the Argonne-AI tenant" width="75%">

Note: these instances are **preconfigured by the admin**. We can deploy additional cluster configurations using Terraform scripts from the public Oracle HPC quickstart repository:

- <https://github.com/oracle-quickstart/oci-hpc>

More details on cluster deployment are covered in the earlier `0_deploying_cluster.md` section.

In this **Argonne-AI test POC**, the baseline configuration includes:

- 1 frontend **login** node  
- 1 **controller** node  
- 2 **GPU compute** nodes  
- 1 **monitoring** node  

Each GPU compute node has:

- 8 × **NVIDIA H200** GPUs  
- 8 × **RDMA-enabled NICs** (NVIDIA/Mellanox ConnectX-7 Ethernet), where `rdma7` is RoCE (RDMA over Converged Ethernet) on top of Ethernet.  

More hardware and topology details are described in `2_oci_compute_infrastructure.md`.

**Slurm** and **LDAP** services are running on the controller node and are used for job scheduling and centralized user management, respectively.

---
### Logging in to the instances from a terminal (SSH)

By default, you are provided with the Linux username **`ubuntu`** on the provisioned instances.

To enable SSH access, you must first share your **public SSH key** with your PI/admin so they can add it to the login node’s `~ubuntu/.ssh/authorized_keys` file.

On your laptop, display your public key (example shown below):

```bash
(base) Kaushik:~ kvelusamy$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDpZtvaDJepMDgbtxpVolTuSmyLbsAO9u4dngs4aMSRqb8tLNE2vVI3VbeKOmi57b78YaSPPTO2fFdr3DpYHx6M1UXSWI6pgwwyDigY2E+RrplNCm2c2uv3j75GZs5TFkNWSlE4qGgRIMK4Pg4yMx6K8qZ5s6BIWxMyPI79FwbvQdG+md/iDbKwjIe6OZNx5UyDa7LV7C/x8GyHyfp4yEBWSiNvzeoni2RzY9f2ucufukScV1hGSo4E9AJuJTNO2DPni3U1vNdQD5xXi3aYY74tFoZWi1gv/b1tcndy1L98AGHoVIiiTXPQTIZ+jucw4psMizio9lNV9KK7BAB06CDDNPTKVSCg+WMqn0+z8vPSRAjsJ03dOaCkStDyxgVDW9Dzf5131HNef5zgF8mtjL7/2uKtR84fo30FLgZsJb56H8xZiPGafXfkHYXIDErwSjQtvrcTgbW9wbuOsDlotZA0GGS+GYz4CNz9AUae0Y/6xpaLVQtto6E= kvelusamy@Kaushik
```

Once the PI/admin has added your public key to one or both of:

- `super-ant-login` (IP: `192.9.140.156`)  
- `super-ant-controller` (IP: `163.192.19.184`)  

under `/home/ubuntu/.ssh/authorized_keys`, you should be able to log in directly using:

```bash
(base) Kaushik:~ kvelusamy$ ssh ubuntu@192.9.140.156
Enter passphrase for key '/Users/kvelusamy/.ssh/id_rsa': 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1074-oracle x86_64)
...
Last login: Wed Nov 26 03:07:29 2025 from 24.12.192.217
ubuntu@super-ant-login:~$
```

You must enter the **passphrase** you set when generating your SSH key. If you used an empty passphrase, just press **Enter**.

---
### Creating a new user on all instances

To create a cluster-wide user account (managed via LDAP and visible on all nodes), SSH to the **controller** node and use the `cluster user` command:

```bash
ubuntu@super-ant-login:~$ ssh super-ant-controller
ubuntu@super-ant-controller:~$ cluster user add kaushik
ubuntu@super-ant-controller:~$
```

This command creates the user on all nodes via LDAP. Because `/home` is a shared filesystem, the home directory is immediately visible across all nodes.

---
### Setting permissions and adding SSH keys for the new user

After the user has been created, log in as that user (e.g., `kaushik`) and fix home-directory permissions and SSH keys as needed:

```bash
# 1) Secure home directory and regular files/directories
kaushik@super-ant-login:~$ chmod 750 "$HOME"          # (or 700 for completely private)
kaushik@super-ant-login:~$ find "$HOME" -type d ! -path "$HOME/.ssh*" -exec chmod 750 {} \;
kaushik@super-ant-login:~$ find "$HOME" -type f ! -path "$HOME/.ssh*" -exec chmod 640 {} \;

# 2) Enforce strict SSH directory and file permissions
kaushik@super-ant-login:~$ mkdir -p "$HOME/.ssh"
kaushik@super-ant-login:~$ chmod 700 "$HOME/.ssh"
kaushik@super-ant-login:~$ touch "$HOME/.ssh/authorized_keys"
kaushik@super-ant-login:~$ chmod 600 "$HOME/.ssh/authorized_keys"

# 3) Extract public key from private key and compare
kaushik@super-ant-login:~$ ssh-keygen -y -f ~/.ssh/id_ed25519 > /tmp/kaushik_id_ed25519.pub
kaushik@super-ant-login:~$ diff /tmp/kaushik_id_ed25519.pub ~/.ssh/authorized_keys || echo "different"

# 4) Append the public key to authorized_keys and re-enforce strict perms
kaushik@super-ant-login:~$ ssh-keygen -y -f ~/.ssh/id_ed25519 >> ~/.ssh/authorized_keys
kaushik@super-ant-login:~$ chmod 700 ~/.ssh
kaushik@super-ant-login:~$ chmod 600 ~/.ssh/authorized_keys

# 5) Test passwordless SSH from login node to a GPU node
kaushik@super-ant-login:~$ ssh GPU-4287
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1074-oracle x86_64)
kaushik@GPU-4287:~$
```

Additional notes:

- Set `umask 027` in `~/.bashrc` if you want new files/directories to default to group-readable, world-inaccessible permissions.
- If you have an existing user that was **not** created via `cluster user` on the controller node, you may need to remove it as `root` and re-create it properly. For example:

  ```bash
  sudo userdel userold
  sudo rm -rf /home/userold
  sudo pkill -KILL -u userold
  ```

  Then re-create the user via `cluster user add`.

---
### Verifying the cluster and logging into GPU nodes

First, list all nodes known to the management tooling:

```bash
kaushik@super-ant-login:~$ mgmt nodes list
┏━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┓
┃ hostname             ┃ healthcheck_recommendation ┃ status  ┃ compute_status ┃ cluster_name ┃ memory_cluster_name ┃ ocid                                                                             ┃ serial        ┃ ip_address    ┃ shape               ┃
┡━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━┩
│ GPU-4287             │ Healthy                    │ running │ configured     │ super-bee    │ None                │ ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyc4z7svanlv6q2da2nmkdadn6plbxfbol… │ 2437XLG0E9    │ 172.16.48.190 │ BM.GPU.H200.8       │
│ super-ant-controller │                            │ running │ configured     │ super-ant    │ None                │ ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyccbcyanj5rgbkkc3kzyig42bmmkpjvik… │ Not Specified │ 172.16.0.225  │ VM.Standard.E5.Flex │
│ super-ant-login      │                            │ running │ configured     │ super-ant    │ None                │ ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpycf4ojecjz3sy6rde232vi2iy2yvp65gh… │ Not Specified │ 172.16.0.59   │ VM.Standard.E5.Flex │
│ GPU-4341             │ Healthy                    │ running │ configured     │ super-bee    │ None                │ ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpych4zt3nl7nv4x65ni4zw4cgmp5jdo5o2… │ 2438XLG0D8    │ 172.16.48.244 │ BM.GPU.H200.8       │
│ super-ant-monitoring │                            │ running │ configured     │ super-ant    │ None                │ ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpycjxfifqh4y6nvk4x3pgunnevbontn76s… │ Not Specified │ 172.16.0.106  │ VM.Standard.E5.Flex │
└──────────────────────┴────────────────────────────┴─────────┴────────────────┴──────────────┴─────────────────────┴──────────────────────────────────────────────────────────────────────────────────┴───────────────┴───────────────┴─────────────────────┘
```

Then, log into a GPU node and verify that the GPU stack is visible and healthy:

```bash
kaushik@super-ant-login:~$ ssh GPU-4287
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-1074-oracle x86_64)
  System load:  0.98                Processes:             2401
  Usage of /:   10.1% of 247.92GB   Users logged in:       1
  Memory usage: 0%                  IPv4 address for eth0: 172.16.48.190
  Swap usage:   0%

Last login: Wed Nov 26 04:25:12 2025 from 172.16.0.59
kaushik@GPU-4287:~$ nvidia-smi
Wed Nov 26 04:35:32 2025   
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.172.08             Driver Version: 570.172.08     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H200                    On  |   00000000:0F:00.0 Off |                    0 |
| N/A   32C    P0            120W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H200                    On  |   00000000:2D:00.0 Off |                    0 |
| N/A   37C    P0            127W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA H200                    On  |   00000000:44:00.0 Off |                    0 |
| N/A   33C    P0            126W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA H200                    On  |   00000000:5B:00.0 Off |                    0 |
| N/A   37C    P0            122W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA H200                    On  |   00000000:89:00.0 Off |                    0 |
| N/A   32C    P0            124W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA H200                    On  |   00000000:A8:00.0 Off |                    0 |
| N/A   37C    P0            127W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA H200                    On  |   00000000:C0:00.0 Off |                    0 |
| N/A   37C    P0            127W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA H200                    On  |   00000000:D8:00.0 Off |                    0 |
| N/A   32C    P0            122W /  700W |       0MiB / 143771MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------------------------------------------------------+
                                                                                  
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
kaushik@GPU-4287:~$
```

At this point, you have:

- Verified that your **OCI web login** and MFA are configured.  
- Confirmed that the **cluster instances** are visible in the Argonne-AI tenant.  
- Established **SSH access** to the login and GPU nodes.  
- Confirmed that **NVIDIA H200 GPUs** and drivers are correctly detected by `nvidia-smi`.

You are now ready to run tests and workloads on the Argonne-AI OCI cluster.
