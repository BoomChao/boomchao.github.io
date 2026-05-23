---
date : '2026-05-23'
lastmod: '2025-05-23'
draft : false
title : 'Golang内存管理'
tags : ["面试题"]
categories: ["golang"]
---
# Go 内存管理

## 计算机内存分布

计算机程序中的数据和变量都存储在内存中。内存中包括两个重要区域：栈和堆。

* 栈主要用于存储函数参数、返回值和局部变量等数据，由编译器进行管理。

* 堆用于存储动态分配的内存，例如对象和数组等，由内存分配器进行分配，由垃圾收集器进行回收。

## 内存管理

从设计原则角度来看，内存管理涉及3部分：用户程序、内存分配器和内存收集器。当用户程序申请内存时，内存分配器申请新的内存。分配器负责从堆中初始化相应的内存区域。然而，内存使用可能成为性能瓶颈，尤其是对于产生大量垃圾的程序

Go语言中的内存分配器包含多个关键组件，包括内存管理单元、线程缓存、中央缓存和页堆；实际在代码层面也主要是下面这四个数据结构

```bash
runtime.msapn
runtime.mcache
runtime.mcentral
runtime.mheap
```

Go 内存架构如下

所有的内存空间都由一个名为 runtime.heapArena 的二维矩阵进行管理

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/golang-mem/images/Go-Mem-diagram.png)

基本逻辑如下：

1. 程序启动会分配一个线程缓存（runtime.mcache）来处理微型对象和小型对象，这些线程缓存持有内存管理单元（runtime.mspan）

2. 每种类型的内存管理单元都管理着特定大小的对象；当内存管理单元中没有空闲对象时就会从全局堆结构（runtime.mheap）中的中央缓存 （runtime.mcentral）中获取新的内存单元

3. 全局堆结构会向操作系统请求内存



### 内存管理单元

内存管理单元是Go语言中管理内存的基本单元，包含两个成员next和prev。这两个成员分别指向下和向上一个相同大小的内存单元，具体如下

```go
type mspan struct {
    next *mspan
    prev *mspan
    ...
}
```

通过这两个指针，同一大小的内存单元构成一个双向链表;

而 runtime.mSpanList 存储这个双向链表的头尾节点，用于线程缓存和中央缓存

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/golang-mem/images/Go-Mem-diagram-1.png)

### 线程缓存

在Go语言中，线程缓存与处理器上的线程绑定，主要用于缓存分配给用户程序的微型对象和小型对象

每个线程缓存有136个内存管理单元，存储在 alloc 内存块中。这些内存管理单元用于管理分配给线程的内存块。使用线程时，直接从缓存的内存管理单元中获取；可参考下图

![](https://raw.githubusercontent.com/BoomChao/boomchao.github.io/main/content/posts/golang-mem/images/Go-Mem-diagram-2.png)

这些内存管理单元用于管理分配给线程的内存块。使用线程时，直接从缓存的内存管理单元中获取管理单元，满足内存分配需求

线程缓存通过中央缓存的cacheSpan方法获取内存管理单元，具体步骤如下

1. 从已清理但含可用空间的 spanSet 结构中获取 runtime.span

2. 从未清理但含空间的 spanSet 结构中获取

3. 从未清理且无空闲空间的 spanSet 中获取,并清理内存空间

4. 通过 Heap 请求新的内存管理单元

5. 更新内存管理单元的 allocCache 等字段，帮助快速分配内存

### 中央缓存

中央缓存是内存分配器的核心，用于管理内存的中央缓存。与线程缓存不同，访问中央缓存中的内存管理单元需要使用互斥锁来确保线程安全，具体如下

```go
type mcentral struct {
    spanclass  spanClass
    partial    [2]spanSet
    full       [2]spanSet
}
```

每个中央缓存管理一个特定大小的内存块，这个内存块被称为跨度类的内存管理单元。这些内存管理单元同时持有runtime.spanSet，用于存储包含空闲对象和不包含空闲对象的内存块

### 页堆

页堆是 Go 语言内存分配的核心结构，用于存储全局变量，用于管理堆上初始化的对象。该结构包含两组非常重要的字段：全局中央缓存列表 central 以及用于管理堆内存区域的 arenas 及其相关字段

一般来说，页堆包含一个长度为136的中央缓存数组，其中68个是需要进行扫描的中央缓存，另外68个是不需要扫描的中央缓存

