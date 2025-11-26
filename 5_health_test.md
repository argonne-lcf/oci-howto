## Health Tests

The **super-ant** cluster includes built-in **GPU and host health checks** that are integrated with Slurm and the `oci-hpc` stack. Health checks:

- Run **every 5 minutes**, and  
- Run at the **start of each job** (when enabled for the partition).

This section shows where the healthcheck scripts live, where logs are written, and how to interpret the output.

---

### 1. Healthcheck scripts under `/opt/oci-hpc/healthchecks`

On each GPU node, the healthcheck scripts and Slurm job files are installed under `/opt/oci-hpc/healthchecks`:

```bash
ubuntu@GPU-4287:~$ ls /opt/oci-hpc/healthchecks/
__pycache__  active_HC.sbatch  active_healthcheck.py  check_gpu_setup.py  gpu_bw_test.py  gpu_sdc_checker.py  multi_node_active_HC.sbatch  multi_node_active_healthcheck.py  rdma_link_flapping.py  shared_logging.py  xid_checker.py
```

Some key components:

- `active_HC.sbatch` / `multi_node_active_HC.sbatch`  
  Slurm job scripts that launch the **active health checks** (single-node and multi-node).

- `active_healthcheck.py` / `multi_node_active_healthcheck.py`  
  Python drivers that orchestrate various GPU/network tests and write results to log files.

- `check_gpu_setup.py`  
  Verifies that the GPU host configuration (drivers, ECC, PCIe, etc.) matches expectations.

- `gpu_bw_test.py`  
  Runs a GPU bandwidth test (often wrapping NCCL or similar microbenchmarks).

- `gpu_sdc_checker.py`  
  Checks for silent data corruption (SDC) patterns on the GPUs.

- `rdma_link_flapping.py`  
  Detects RDMA link flaps or downed links.

- `xid_checker.py`  
  Looks for NVIDIA XID (GPU error) events.

All these tests feed into the healthcheck logs described in the next section.

---

### 2. Healthcheck logs under `/var/log/healthchecks`

Each node maintains a set of healthcheck logs under `/var/log/healthchecks`:

```bash
ubuntu@GPU-4287:~$ cd /var/log/healthchecks/

ubuntu@GPU-4287:/var/log/healthchecks$ ls
active_HealthCheck  latest_active_healthcheck.log  latest_healthcheck.log
```

The key files:

- `latest_healthcheck.log`  
  Summary of the most recent **host setup / configuration** check on this node.

- `latest_active_healthcheck.log`  
  Summary of the most recent **active GPU health check**, including GPU/NCCL tests.

- `active_HealthCheck/`  
  Directory containing per-run output files (e.g., one file per Slurm healthcheck job).

#### 2.1 Example active healthcheck summary log

```bash
ubuntu@GPU-4287:/var/log/healthchecks$ cat latest_active_healthcheck.log 
Started GPU active healthcheck at: 2025-11-26-071550
Node details: GPU-4287 - BM Instance: 2437XLG0E9 - ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyc4z7svanlv6q2da2nmkdadn6plbxfbolfl3tqhmqg62la - BM.GPU.H200.8
```

This confirms:

- The timestamp when the active healthcheck started.
- The node name (`GPU-4287`), bare-metal instance ID (`2437XLG0E9`), OCID, and shape (`BM.GPU.H200.8`).

#### 2.2 Example host setup healthcheck log

```bash
ubuntu@GPU-4287:/var/log/healthchecks$ cat latest_healthcheck.log 
Started GPU host setup check at: 2025-11-26-071542
Node details: GPU-4287 - BM Instance: 2437XLG0E9 - BM.GPU.H200.8
Node details: ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyc4z7svanlv6q2da2nmkdadn6plbxfbolfl3tqhmqg62la
OCA state is: COMPLETED
Oracle Cloud Agent: 1.52.0
RTTCC Disabled Check: Passed
GPU ECC Test: Passed
GPU Row Remap Test: Passed
GPU Count Test: Passed
GPU PCIe Width Test: Passed
Bus Check Test: Passed
RDMA Link Status Check: Passed
RDMA link flapping/down test: Passed
Xid Check: Passed
WPA Authentication Check: Passed
Fabric Manager Running: Passed
CPU Profile Check: Passed
All interfaces have an IP defined: Passed
For 8 GPUs, 18 links detected as expected and all speeds as expected 25 GB/s : Passed
dcgmi health check: Passed
--------- Summary of Host setup check for BM Instance: 2437XLG0E9 ---------
Finished GPU host setup check at: 2025-11-26-071542
```

Highlights:

- **OCA state** – Confirms Oracle Cloud Agent is in the expected state.
- **GPU ECC / Row Remap / Count / PCIe Width / Bus Check** – Verifies that all 8 H200 GPUs have the correct reliability configuration and that PCIe lanes are as expected.
- **RDMA link status / link flapping** – Validates that RoCE/IB links are up and stable.
- **Xid Check** – Ensures there are no NVIDIA XID errors.
- **Fabric Manager Running** – Confirms NVIDIA Fabric Manager is operational.
- **Network checks** – Confirms per-interface IP configuration and link speeds (18 links at 25 GB/s for 8 GPUs).
- **`dcgmi health check`** – Wraps NVIDIA DCGM health checks for GPUs.

If any of these checks fail, they will be reported as `Failed` with more details, and the node may be marked unhealthy in Slurm/management tooling.

---

### 3. Per-run active healthcheck outputs

For each active healthcheck run, a more detailed log is stored in the `active_HealthCheck` directory. For example:

```bash
ubuntu@GPU-4287:/var/log/healthchecks$ cd active_HealthCheck/

ubuntu@GPU-4287:/var/log/healthchecks/active_HealthCheck$ cat  active_HealthCheck-11298.out
2025-11-26 07:15:50,350 - INFO - Started GPU active healthcheck at: 2025-11-26-071550
2025-11-26 07:15:50,351 - INFO - Node details: GPU-4287 - BM Instance: 2437XLG0E9 - ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyc4z7svanlv6q2da2nmkdadn6plbxfbolfl3tqhmqg62la - BM.GPU.H200.8
2025-11-26 07:16:15,062 - INFO - GPU-4287 - NCCL Test Succeeded: NCCL Test Succeeded: Avg bus bandwidth is 475.815
```

Here we see:

- The same start time and node identification as in `latest_active_healthcheck.log`.
- An **NCCL functional and performance test**:
  - `NCCL Test Succeeded` – Confirms that NCCL multi-GPU communication worked correctly.
  - `Avg bus bandwidth is 475.815` – Confirms high-bandwidth performance for the active check.

These per-run logs are especially useful when debugging intermittent issues or correlating a failed job with a specific healthcheck run (e.g., by job ID in the filename).

---

### 4. Health status via management tooling

In addition to reading logs directly, the **management CLI** can surface health information. For example, the `mgmt nodes get <node>` command (used elsewhere in this guide) includes health-related fields derived from these checks.

You can use:

- `mgmt nodes list` – to see overall node status and health recommendations.
- `mgmt nodes get GPU-4287` – to retrieve detailed information for a specific node, including results of recent health checks.

This provides a cluster-wide summary view, while `/var/log/healthchecks` contains the detailed, per-node logs.

---

With these tools and logs, you can:

- Confirm that GPU nodes are correctly configured.
- Verify that the H200 GPUs and RDMA links are healthy and performant.
- Quickly diagnose health-related failures or performance anomalies before running large-scale workloads.
