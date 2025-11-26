


```
ubuntu@GPU-4287:~$ ls /opt/oci-hpc/healthchecks/
__pycache__  active_HC.sbatch  active_healthcheck.py  check_gpu_setup.py  gpu_bw_test.py  gpu_sdc_checker.py  multi_node_active_HC.sbatch  multi_node_active_healthcheck.py  rdma_link_flapping.py  shared_logging.py  xid_checker.py
ubuntu@GPU-4287:~$ 
ubuntu@GPU-4287:~$ cd /var/log/healthchecks/
ubuntu@GPU-4287:/var/log/healthchecks$ ls
active_HealthCheck  latest_active_healthcheck.log  latest_healthcheck.log
ubuntu@GPU-4287:/var/log/healthchecks$ cat latest_active_healthcheck.log 
Started GPU active healthcheck at: 2025-11-26-071550
Node details: GPU-4287 - BM Instance: 2437XLG0E9 - ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyc4z7svanlv6q2da2nmkdadn6plbxfbolfl3tqhmqg62la - BM.GPU.H200.8
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
ubuntu@GPU-4287:/var/log/healthchecks$ cd active_HealthCheck/

ubuntu@GPU-4287:/var/log/healthchecks/active_HealthCheck$ cat  active_HealthCheck-11298.out
2025-11-26 07:15:50,350 - INFO - Started GPU active healthcheck at: 2025-11-26-071550
2025-11-26 07:15:50,351 - INFO - Node details: GPU-4287 - BM Instance: 2437XLG0E9 - ocid1.instance.oc1.us-sanjose-1.anzwuljrhhkgvpyc4z7svanlv6q2da2nmkdadn6plbxfbolfl3tqhmqg62la - BM.GPU.H200.8
2025-11-26 07:16:15,062 - INFO - GPU-4287 - NCCL Test Succeeded: NCCL Test Succeeded: Avg bus bandwidth is 475.815


```
