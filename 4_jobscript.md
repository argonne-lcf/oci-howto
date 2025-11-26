## Jobscript

We use slurm as provisioned during the cluster deployment. Below is a sample example to verify and run simple jobscripts.

```
ubuntu@super-ant-controller:~$ sinfo 
PARTITION           AVAIL  TIMELIMIT  NODES  STATE NODELIST
compute*               up   infinite      1  alloc GPU-4287
compute*               up   infinite      1   idle GPU-4341
compute-healthcheck    up   infinite      1  alloc GPU-4287
compute-healthcheck    up   infinite      1   idle GPU-4341
ubuntu@super-ant-controller:~$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
             11235 compute-h active_H   ubuntu  R       0:35      1 GPU-4287
ubuntu@super-ant-controller:~$ 
```

```
ubuntu@super-ant-controller:~$ ls /opt/oci-hpc/samples/gpu/*
/opt/oci-hpc/samples/gpu/H100-topology-kubernetes.xml                        /opt/oci-hpc/samples/gpu/nccl_run_allreduce_srun.sbatch          /opt/oci-hpc/samples/gpu/qfabv1_nccl_run_allreduce.sbatch
/opt/oci-hpc/samples/gpu/H100-topology.xml                                   /opt/oci-hpc/samples/gpu/nccl_run_allreduce_srun.sh              /opt/oci-hpc/samples/gpu/qfabv1_nccl_run_allreduce.sh
/opt/oci-hpc/samples/gpu/ifdown.sh                                           /opt/oci-hpc/samples/gpu/nccl_run_allreduce_tuner.sbatch         /opt/oci-hpc/samples/gpu/qfabv1_nccl_run_alltoall.sh
/opt/oci-hpc/samples/gpu/ifup.sh                                             /opt/oci-hpc/samples/gpu/nccl_run_allreduce_tuner.sh             /opt/oci-hpc/samples/gpu/rccl_run_allreduce.sbatch
/opt/oci-hpc/samples/gpu/nccl_run_allreduce.sbatch                           /opt/oci-hpc/samples/gpu/nccl_run_alltoall.sh                    /opt/oci-hpc/samples/gpu/srun_examples_with_container.txt
/opt/oci-hpc/samples/gpu/nccl_run_allreduce.sh                               /opt/oci-hpc/samples/gpu/no_ncclparam_nccl_run_allreduce.sbatch  /opt/oci-hpc/samples/gpu/topo-flattened-b4.xml
/opt/oci-hpc/samples/gpu/nccl_run_allreduce_GB200.sbatch                     /opt/oci-hpc/samples/gpu/no_ncclparam_nccl_run_allreduce.sh      /opt/oci-hpc/samples/gpu/topo-flattened.xml
/opt/oci-hpc/samples/gpu/nccl_run_allreduce_containers_H100_H200.sbatch      /opt/oci-hpc/samples/gpu/notes.txt                               /opt/oci-hpc/samples/gpu/update_arp_settings.sh
/opt/oci-hpc/samples/gpu/nccl_run_allreduce_containers_with_ordering.sbatch  /opt/oci-hpc/samples/gpu/ping.sh                                 /opt/oci-hpc/samples/gpu/update_netmask.sh
ubuntu@super-ant-controller:~$ 
```

```
kaushik@super-ant-controller:/opt/oci-hpc/samples$ cat NCCL_readme 
To Run a NCCL test, run the following commands: 
chmod 775 /opt/oci-hpc/samples/prep_sample_files.sh
/opt/oci-hpc/samples/prep_sample_files.sh

SSH to one of the compute nodes and run: ~/compile.sh

From the controller, you can edit the third line of /home/opc/nccl_run_allreduce.sbatch with the number of nodes that you would like to test on:
sbatch /home/opc/nccl_run_allreduce.sbatch

Look at the last line of the log for bandwidth. 

kaushik@super-ant-controller:/opt/oci-hpc/samples$ ./prep_sample_files.sh 
kaushik@super-ant-controller:~$ sbatch nccl_run_allreduce.sbatch 
Submitted batch job 11286
kaushik@super-ant-controller:~$ head nccl_run_allreduce.sbatch 
#!/bin/bash
#SBATCH --job-name=nccl-allreduce-slurm
#SBATCH --nodes=2
#SBATCH --gpus-per-node=8
#SBATCH --ntasks-per-node=8
#SBATCH --exclusive
export PMI_DEBUG=1
kaushik@super-ant-controller:~$ cat slurm.out

# This script defaults to running all_reduce_perf, but can run other NCCL tests if launched with
# EXEC=all_gather_perf sbatch -N8 ./nccl_run_allreduce_H100_200.sbatch

MACHINEFILE
GPU-6550
GPU-3841
ORDEREDRANKMACHINEFILE
rank 0=GPU-6550 slot=0
rank 1=GPU-6550 slot=1
rank 2=GPU-6550 slot=2
rank 3=GPU-6550 slot=3
rank 4=GPU-6550 slot=4
rank 5=GPU-6550 slot=5
rank 6=GPU-6550 slot=6
rank 7=GPU-6550 slot=7
rank 8=GPU-3841 slot=0
rank 9=GPU-3841 slot=1
rank 10=GPU-3841 slot=2
rank 11=GPU-3841 slot=3
rank 12=GPU-3841 slot=4
rank 13=GPU-3841 slot=5
rank 14=GPU-3841 slot=6
rank 15=GPU-3841 slot=7
NODE_SWITCH_LIST
Node GPU-6550 from switch number 1
Node GPU-3841 from switch number 1
Running /opt/oci-hpc/nccl-test/build/all_reduce_perf test on 2 nodes
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 17179869184 step: 2(factor) warmup iters: 5 iters: 50 agg iters: 1 validation: 1 graph: 0
#
# Using devices
#  Rank  0 Group  0 Pid 320119 on   GPU-6550 device  0 [0x0f] NVIDIA H200
#  Rank  1 Group  0 Pid 320120 on   GPU-6550 device  1 [0x2d] NVIDIA H200
#  Rank  2 Group  0 Pid 320121 on   GPU-6550 device  2 [0x44] NVIDIA H200
#  Rank  3 Group  0 Pid 320122 on   GPU-6550 device  3 [0x5b] NVIDIA H200
#  Rank  4 Group  0 Pid 320123 on   GPU-6550 device  4 [0x89] NVIDIA H200
#  Rank  5 Group  0 Pid 320124 on   GPU-6550 device  5 [0xa8] NVIDIA H200
#  Rank  6 Group  0 Pid 320125 on   GPU-6550 device  6 [0xc0] NVIDIA H200
#  Rank  7 Group  0 Pid 320126 on   GPU-6550 device  7 [0xd8] NVIDIA H200
#  Rank  8 Group  0 Pid 287770 on   GPU-3841 device  0 [0x0f] NVIDIA H200
#  Rank  9 Group  0 Pid 287771 on   GPU-3841 device  1 [0x2d] NVIDIA H200
#  Rank 10 Group  0 Pid 287772 on   GPU-3841 device  2 [0x44] NVIDIA H200
#  Rank 11 Group  0 Pid 287773 on   GPU-3841 device  3 [0x5b] NVIDIA H200
#  Rank 12 Group  0 Pid 287774 on   GPU-3841 device  4 [0x89] NVIDIA H200
#  Rank 13 Group  0 Pid 287775 on   GPU-3841 device  5 [0xa8] NVIDIA H200
#  Rank 14 Group  0 Pid 287776 on   GPU-3841 device  6 [0xc0] NVIDIA H200
#  Rank 15 Group  0 Pid 287777 on   GPU-3841 device  7 [0xd8] NVIDIA H200
NCCL version 2.27.5+cuda12.8
#
#                                                              out-of-place                       in-place        
#       size         count      type   redop    root     time   algbw   busbw #wrong     time   algbw   busbw #wrong
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)     
  1073741824     268435456     float     sum      -1   4396.8  244.21  457.90      0   4382.1  245.03  459.43      0
  2147483648     536870912     float     sum      -1   8538.8  251.50  471.56      0   8500.0  252.64  473.71      0
  4294967296    1073741824     float     sum      -1    16756  256.32  480.61      0    16759  256.28  480.53      0
  8589934592    2147483648     float     sum      -1    33250  258.34  484.39      0    33228  258.51  484.71      0
 17179869184    4294967296     float     sum      -1    66214  259.46  486.49      0    66226  259.41  486.40      0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 476.572 
#

```
