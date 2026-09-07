---
date : '2026-09-07T19:00:00+08:00'
draft : false
title : '容器里的 /dev/shm：为什么推理服务要挂 1T 的内存卷'
tags : ["kubernetes", "容器", "推理"]
categories: ["kubernetes"]
---

今天观察线上的推理服务，发现一个很有意思的现象，看到一个卷的挂载是这样的

```yaml
volumeMounts:
- mountPath: /dev/shm
  name: dshm
volumes:
- emptyDir:
    medium: Memory
    sizeLimit: 1280Gi
  name: dshm
```

可以看到这个 emptyDri 申请的居然是内存而不是磁盘，而且是约 1个 T 左右的内存，并且是挂载到容器的 /dev/shm 这个路径下面的

想要理解这个用法我们需要铺垫点知识

首先 /dev/shm 是什么？

`/dev/shm` 是 Linux 的共享内存设备，是一个由内核 tmpfs 提供的内存文件系统

shm = shared memory（共享内存），它是 POSIX 共享内存的挂载点；本质是 tmpfs，文件全在 RAM 内存里面，不落盘，断电即失；其默认大小一般是物理内存的一半

```bash
root@10-39-32-156:~# df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            31G  120K   31G   1% /dev/shm
root@10-39-32-156:~# free -h
               total        used        free      shared  buff/cache   available
Mem:            60Gi        10Gi       5.5Gi        15Mi        45Gi        50Gi
Swap:             0B          0B          0B
```

比如我这台机器物理内存就 60G，然后共享内存分配占了 30G

而容器的大小默认是 64M，随便到一个k8s集群的容器里面去查看这个大小，查看如下

```bash
$ df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
shm              64M     0   64M   0% /dev/shm
```

推理为什么要调大这个东西？

推理服务需要多 worker 之间用共享内存传 tensor、NCCL/vLLM 多进程通信都要走 /dev/shm，64MB 直接报 Bus error / invalid argument；所以训练/推理 pod 普遍要挂一个大shm——把 K8s 默认的 64MB 换成 1280Gi
