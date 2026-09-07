---
date : '2026-09-07T19:00:00+08:00'
draft : false
title : '容器里的 /dev/shm：为什么推理服务要挂 1T 的内存卷'
tags : ["kubernetes", "容器", "推理"]
categories: ["kubernetes"]
---

# 从一个有意思的挂载说起

今天观察线上的推理服务，发现一个很有意思的现象，看到一个卷的挂载是这样的：

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

可以看到这个 emptyDir 申请的居然是**内存**而不是磁盘，而且是约 1 个 T 左右的内存，并且挂载到了容器的 `/dev/shm` 这个路径下面。

想要理解这个用法，我们需要先铺垫一点背景知识。

# /dev/shm 是什么？

`/dev/shm` 是 Linux 的共享内存设备，是一个由内核 tmpfs 提供的内存文件系统。

shm = shared memory（共享内存），它是 POSIX 共享内存的挂载点。本质是 tmpfs，文件全在 RAM 内存里面，不落盘，断电即失；其默认大小一般是物理内存的一半。

```bash
root@10-39-32-156:~# df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            31G  120K   31G   1% /dev/shm
root@10-39-32-156:~# free -h
               total        used        free      shared  buff/cache   available
Mem:            60Gi        10Gi       5.5Gi        15Mi        45Gi        50Gi
Swap:             0B          0B          0B
```

比如我这台机器物理内存是 60G，然后共享内存默认就占了 31G，差不多就是物理内存的一半。

# 容器里的 /dev/shm 有多大？

宿主机上是内存的一半，但到了容器里情况就不一样了——**容器里的 /dev/shm 默认只有 64M**。

随便到一个 K8s 集群的容器里面去查看这个大小：

```bash
$ df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
shm              64M     0   64M   0% /dev/shm
```

这是因为容器运行时（Docker / containerd）在创建容器时，默认会给 `/dev/shm` 挂一个 64MB 的 shm 设备，这个默认值对绝大多数普通应用是够用的。

# 推理服务为什么要调大它？

问题在于，推理服务不是"普通应用"。

推理服务需要多 worker 之间用共享内存传 tensor，NCCL / vLLM 的多进程通信都要走 `/dev/shm`。这些场景下传输的是动辄几百 MB 甚至几个 G 的 tensor 数据，64MB 的默认大小根本塞不下，直接就会报 `Bus error` / `invalid argument` 这类错误。

所以训练/推理 pod 普遍要挂一个大 shm——把 K8s 默认的 64MB 换成文章开头的 1280Gi。

# K8s 里怎么做？

做法就是开头看到的那个配置：用 `emptyDir` + `medium: Memory` 挂一个内存卷到 `/dev/shm`，把容器运行时默认的 64MB shm 覆盖掉：

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

两个关键点：

1. `medium: Memory` 告诉 kubelet 这个 emptyDir 走 tmpfs（内存）而不是节点磁盘，读写性能最好，正好匹配共享内存通信的诉求；
2. `sizeLimit: 1280Gi` 把这个 tmpfs 的上限调到 1T+，够多 worker 之间传大 tensor 用了。注意这部分内存是计入容器内存使用的，所以 Pod 的 memory request/limit 也要相应留足，否则会被 OOMKill。

# 总结

- `/dev/shm` 是 tmpfs 实现的 POSIX 共享内存，宿主机上默认是物理内存的一半；
- 容器里默认只有 64MB，对普通应用够用，但对 NCCL / vLLM 这类要走共享内存传 tensor 的推理服务完全不够；
- K8s 里的标准解法：`emptyDir` + `medium: Memory` 挂到 `/dev/shm`，并用 `sizeLimit` 调大上限。

下次再看到推理 pod 里挂一个 1T 的"内存盘"，就知道它在干什么了。
