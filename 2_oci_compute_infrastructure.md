### installing clush on login node.

Initially OCI did not have `clush` or `pdsh` on the login node, so i installed it. These help facilitate various cluster-wide user-task management. 

```
ubuntu@super-ant-login:~$ sudo apt-get install -y clustershell
```

### Verfying the memory and filesystem

The GPU compute nodes have roughly 3 TB of DRAM.

```
kaushik@super-ant-login:~$ clush  -w 172.16.48.190,172.16.0.225,172.16.0.59,172.16.48.244 'hostname' 
172.16.0.225: super-ant-controller
172.16.0.59: super-ant-login
172.16.48.190: GPU-4287
172.16.48.244: GPU-4341
kaushik@super-ant-login:~$ 

kaushik@super-ant-login:~$ clush  -w 172.16.48.190,172.16.0.225,172.16.0.59,172.16.48.244 'free -g' 
172.16.0.225:                total        used        free      shared  buff/cache   available
172.16.0.225: Mem:             125           1         115           0           8         123
172.16.0.225: Swap:              0           0           0
172.16.0.59:                total        used        free      shared  buff/cache   available
172.16.0.59: Mem:             251           0          88           0         162         248
172.16.0.59: Swap:              0           0           0
172.16.48.190:                total        used        free      shared  buff/cache   available
172.16.48.190: Mem:            3023          24        2988           0          10        2987
172.16.48.190: Swap:              0           0           0
172.16.48.244:                total        used        free      shared  buff/cache   available
172.16.48.244: Mem:            3023          21        2969           0          32        2989
172.16.48.244: Swap:              0           0           0
kaushik@super-ant-login:~$ clush  -w 172.16.48.190,172.16.0.225,172.16.0.59,172.16.48.244 'hostname' 
172.16.0.59: super-ant-login
172.16.0.225: super-ant-controller
172.16.48.190: GPU-4287
172.16.48.244: GPU-4341
```

All nodes have a common shared file system /config, /home and /fss and varying sizes of local SSDs.

Note the common shared file system is NFS and not lustre

```
kaushik@super-ant-login:~$ clush  -w 172.16.48.190,172.16.0.225,172.16.0.59,172.16.48.244 'df -h' 
172.16.0.225: Filesystem                                        Size  Used Avail Use% Mounted on
172.16.0.225: tmpfs                                              13G  1.5M   13G   1% /run
172.16.0.225: /dev/sda1                                         993G   24G  969G   3% /
172.16.0.225: tmpfs                                              63G     0   63G   0% /dev/shm
172.16.0.225: tmpfs                                             5.0M     0  5.0M   0% /run/lock
172.16.0.225: /dev/sda15                                        105M  6.1M   99M   6% /boot/efi
172.16.0.225: fss-super-ant-controller.super-ant.local:/config  8.0E  1.8G  8.0E   1% /config
172.16.0.225: fss-super-ant-controller.super-ant.local:/home    8.0E   35G  8.0E   1% /home
172.16.0.225: fss-super-ant-controller.super-ant.local:/fss     8.0E  137G  8.0E   1% /fss
172.16.0.225: tmpfs                                              13G  4.0K   13G   1% /run/user/10001
172.16.0.59: Filesystem                                      Size  Used Avail Use% Mounted on
172.16.0.59: tmpfs                                            26G  1.7M   26G   1% /run
172.16.0.59: /dev/sda1                                       497G   23G  474G   5% /
172.16.0.59: tmpfs                                           126G     0  126G   0% /dev/shm
172.16.0.59: tmpfs                                           5.0M     0  5.0M   0% /run/lock
172.16.0.59: /dev/sda15                                      105M  6.1M   99M   6% /boot/efi
172.16.0.59: fss-super-ant-controller:/config                8.0E  1.8G  8.0E   1% /config
172.16.0.59: fss-super-ant-controller.super-ant.local:/home  8.0E   35G  8.0E   1% /home
172.16.0.59: fss-super-ant-controller.super-ant.local:/fss   8.0E  137G  8.0E   1% /fss
172.16.0.59: tmpfs                                            26G  8.0K   26G   1% /run/user/1001
172.16.0.59: tmpfs                                            26G  8.0K   26G   1% /run/user/10001
172.16.48.190: Filesystem                                      Size  Used Avail Use% Mounted on
172.16.48.190: tmpfs                                           303G  5.3M  303G   1% /run
172.16.48.190: /dev/sda1                                       248G   26G  223G  11% /
172.16.48.190: tmpfs                                           1.5T     0  1.5T   0% /dev/shm
172.16.48.190: tmpfs                                           5.0M     0  5.0M   0% /run/lock
172.16.48.190: /dev/sda15                                      105M  6.1M   99M   6% /boot/efi
172.16.48.190: fss-super-ant-controller:/config                8.0E  1.8G  8.0E   1% /config
172.16.48.190: /dev/md0                                         28T  200G   28T   1% /mnt/localdisk
172.16.48.190: fss-super-ant-controller.super-ant.local:/home  8.0E   35G  8.0E   1% /home
172.16.48.190: fss-super-ant-controller.super-ant.local:/fss   8.0E  137G  8.0E   1% /fss
172.16.48.190: tmpfs                                           303G  8.0K  303G   1% /run/user/1001
172.16.48.190: tmpfs                                           303G  8.0K  303G   1% /run/user/10001
172.16.48.244: Filesystem                                      Size  Used Avail Use% Mounted on
172.16.48.244: tmpfs                                           303G  5.3M  303G   1% /run
172.16.48.244: /dev/sda1                                       248G   26G  223G  11% /
172.16.48.244: tmpfs                                           1.5T     0  1.5T   0% /dev/shm
172.16.48.244: tmpfs                                           5.0M     0  5.0M   0% /run/lock
172.16.48.244: /dev/sda15                                      105M  6.1M   99M   6% /boot/efi
172.16.48.244: fss-super-ant-controller:/config                8.0E  1.8G  8.0E   1% /config
172.16.48.244: /dev/md0                                         28T  200G   28T   1% /mnt/localdisk
172.16.48.244: fss-super-ant-controller.super-ant.local:/home  8.0E   35G  8.0E   1% /home
172.16.48.244: fss-super-ant-controller.super-ant.local:/fss   8.0E  137G  8.0E   1% /fss
172.16.48.244: tmpfs                                           303G  8.0K  303G   1% /run/user/10001
kaushik@super-ant-login:~$ 

```

### Verfying the network interfaces


```
ubuntu@GPU-4287:~$ ibdev2netdev 
mlx5_0 port 1 ==> rdma0 (Up)
mlx5_1 port 1 ==> eth0 (Up)
mlx5_10 port 1 ==> rdma6 (Up)
mlx5_11 port 1 ==> rdma7 (Up)
mlx5_2 port 1 ==> eth1 (Down)
mlx5_3 port 1 ==> rdma1 (Up)
mlx5_4 port 1 ==> rdma2 (Up)
mlx5_5 port 1 ==> rdma3 (Up)
mlx5_6 port 1 ==> rdma4 (Up)
mlx5_7 port 1 ==> eth2 (Up)
mlx5_8 port 1 ==> eth3 (Down)
mlx5_9 port 1 ==> rdma5 (Up)
ubuntu@GPU-4287:~$ 
```


It is better to copy the datasets from the NFS to the local SSDs for performance. 
NFS is roughly 8GiB/s peak. 

You can also start services like vector databases through ansible under image oci-hpc playbooks or under /config/playbooks/roles/custom/tasks/
