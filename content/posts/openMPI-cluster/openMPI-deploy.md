---
title: "Deployment of openMPI VM cluster"
authors: xen134
description: VM image for openMPI study.
summary: VM image for openMPI study.
date: 2021-09-24T13:54:08+08:00
draft: false
---

#### Preparation

Note that below hardware requirements are not compulsory. It is possible to run the cluster on a host with lower performance, however the experience may not be good.

1. CPU with cores more than 8. Each node will have 4 cores by default.

2. Machine with at least 16 GB RAM, 32 Gb or more is preferable. By default one node will be assigned with 4 GB memory.

3. Free disk space 10 GB +, 50 - 100 GB is preferable. One empty node is about 4 GB in size.

4. VirtualBox installation.

5. Download the MPI-base and MPI-worker ova files.

   Download from MEGA: [debian-MPI-base.ova](https://mega.nz/file/GoEXFQwK#B_T6KyuIUF9kfwmouRTgs74lIxRMggVbwL4HSG6bgWU) and [debian-MPI-worker-1.ova](https://mega.nz/file/agMnxQST#oPt-VJhTh3Yh-aE38WcdIep8hIhZnDhbvHQ79izWsmU)

#### Deployment

1. Import the MPI-base to your VirtualBox by simply double clicking the ova file. You may adjust the system settings at the import dialog.

2. Import the MPI-worker to the VirtualBox following the same routine.

3. Boot the imported VM, both the base and the worker. Check their IP addresses on the **Internal network** and edit the corresponding entries in `/etc/hosts`. 

   For base, the `/etc/hosts` should contain all worker nodes. Remove non-existing workers and assign existing workers with unique names by specifying their IP.

   For worker, the `/etc/hosts` should only contain the base node, which is named manager. Replace the IP of manager with the real IP in your virtual network.

4. Then a cluster with one manager and one worker is deployed. Directory `/home/xen134/cloud` is shared via NFS between manager and workers. You can put your code and binary here for MPI run.

   Test the cluster by executing 

   ```shell
   $ cd /home/xen134/cloud
   $ mpirun -np 8 -host localhost:4,worker1:4 ./a.out
   ```

   Then both the manager and the worker will have 4 processes running `a.out`.

5. If you want to add nodes, simply clone the worker in VirtualBox. Remember to choose  'Generate new MAC addresses for all network adapters' when cloning in case of collision. After cloning repeat step 2 - step 4. 
