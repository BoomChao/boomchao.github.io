---
date : '2026-09-18T09:00:00+08:00'
draft : false
title : 'GPU架构初探'
tags : ["GPU", "NVIDIA", "H100", "硬件架构"]
categories: ["基础设施"]
---

GPU 总体设计如下这张图（以 Nvidia H100 为例）

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gpu/images/GPU-image.png)

Nvidia H100 GPU的核心芯片是Nvidia GH100；NvidiaGH100对外的接口有16个PCI-E 5.0通道、9个NVLink 4.0通道和6个HBM 3/HBM 2e通道

GPU 在本质上是一个PCI-E插卡/扣卡，由 PCB（Printed CircultBoard，印刷电路板）、GPU芯片、GPU内存（或称“显存”​）及其他附属电路构成

从上面这个图就可以看到，NVLink 就是和 GPU 通信的，PCI-E 就是和 CPU 通信的

HBM 全称就是 High Bandwidth Memory（高带宽内存也就是显存）

PCI-E 5.0 就理解为一种通信标准（5.0 代表第 5 代）

介绍一个新的概念，NVLink Switch：用于实现GPU之间的互访，让一个GPU可以在CPU无感知的情况下访问另一个GPU的内存，而无须绕行PCI-E总线

注意区分 NVLink 和 NVLink Switch 的关系；可以一句话形容：NVLink 是网线，NVSwitch 是交换机

* NVLink 点对点连接两块芯片

* NVLink Switch 把 GPU 的 NVLink 汇成交换网络，实现全互联

弄清楚 HBM，NVLink 和 PCI-E 之后，我们深入看下 GH100 芯片部分如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gpu/images/GPU-image-1.png)

整个Nvidia GH100芯片有8个GPC（GPU Processing Cluster，GPU处理集群）​，每4个GPC都共用30MB的L2 Cache（二级缓存）​，每个GPC都有9个TPC（Texture Processing Cluster，纹理处理集群）​，在每个TPC内都有2个SM。也就是说，整颗Nvidia GH100芯片集成了144个SM

SM 全称是 Streaming Multiprocessor（流式多处理器），这是 GPU 芯片的核心部件；SM 的内部图如下

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/gpu/images/GPU-image-2.png)

在每个SM内部都有256KB的L1Data Cache（一级数据缓存）​，被所有计算单元共享，同时，在SM内部还有4个纹理处理单元Tex

SM的计算核心部件为Tensor Core和CUDA Core（由图中的 INT32单元、2个FP32单元和1个FP64计算单元组成）

在 Hopper 架构中，每个SM都有4个象限，每个象限都包含1个TensorCore和32个CUDA Core，总计4个Tensor Core和128个CUDA Core。整颗芯片可用的CUDA Core数量为144×128=18432个，可用的TensorCore数量为144×4=576个

理解计算单元和缓存，在 Hopper 架构之下，访问速度从高到低依次如下

* 访问速度最快的是SM中每个象限的1KB RegisterFile

* 访问速度次之的是每个象限的1块L0指令缓存，被32个CUDA Core和1个Tensor Core共用

* 访问速度更慢一些的，是每个SM中的256KB L1 Data Cache，由所有CUDA Core和Tensor Core共用

* 比L1 Data Cache更慢的是在整颗芯片中集成的60MB的L2 Cache，由2个BANK组成，最慢的是Nvidia GH100芯片外部的HBM3显存

所以你现在应该明白了GPU之所以能够用来支撑机器学习程序的高效运行，其**根本原因是GPU内部集成了大量的通用计算单元（如CUDA Core）和专用计算单元（如Tensor Core）**



区分这两个核心：Tensor Core 和 Cuda Core

CUDA Core 是通用的标量运算单元，Tensor Core 是矩阵乘法专用单元；前者什么都能算但是一次只算一个数，后者只会矩阵乘，但一次一整块

所以GPU 的深度学习算力看的从来都不是 CUDA Core的数量，而是 Tensor Core 的数量



## 参考资料

书籍《大模型时代的基础架构：大模型算力中心建设指南》
