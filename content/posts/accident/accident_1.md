---
date : '2025-03-10T23:42:12+08:00'
draft : true
title : '记一起严重的线上事故总结'
tags : ["事故总结"]
categories: ["事故总结"]
---

最近组内发生了一起严重的事故，涉及的影响面非常广，组内后面也进行了深刻的复盘，我本人也经历了
这次事故的一个完整的流程，于是想着记录点什么能够避免后面此种类型的事故，提高稳定性


## 业务介绍
组内是做云原生业务，负责托管业务服务，并且能够提供一些弹性调度和资源管理以及服务监控的功能，
也就是客户把服务托管在平台上，就可以做到整个链路的调用检测以及可以弹性根据服务的特性设置最大和最小
副本数以及扩所容的负载阈值等等

TODO: 这里要贴一张平台整体的架构图


## 事故描述
凌晨12:40～00:30，某个kt服务突发疯狂性的自动扩容，扩容的容器数量从400->3400个；
而且导致将近100台宿主机发生了内存级别的OOM,这些机器的相关的CPU和内存利用率等prometheus采集指标在这个时间段丢失
更严重的是和这个服务部署在同一台机器的服务也收到了影响，大量的oncall工单发起，平台的工单半个小时
达到了将近30起，基本都是服务主调或者被调的延时上升或者失败率增加很多

## 事故止损
未有相关的止损措施，当我们发现并且接入排查问题时，业务的扩容高峰已经过去，事故已经自动终止，没有造成更近一步的损害

## 事故原因
kt服务由于内存泄露，导致服务在短时间内疯狂消费内存，短时间内达到了机器的高负载阈值，所以服务疯狂进行了扩容，由于这个
服务原来的pod数量就很多，导致扩容的数量成倍数增加; 但是因为平台的一些策略，导致扩容的时候很多pod进行了同机部署，这就导致
扩容成功后还有问题，pod短时间内打满CPU达到OOM，也进一步将机器的CPU打满，导致机器的CPU跑满进而影响了同机上的其他pod运作

## 事故教训
1. 没有及时的告警通知知会到平台的相关人员，事故发生了居然还是通过其他oncall的工单平台相关人员才知道
2. 调度机器在容器扩容的时候没有打散，导致多个pod部署到了同一个node上面，这些pod同时有问题导致宿主机发生
问题，进行影响到了其他的服务
3. 提前没有退水的处理措施，如果这个事故发生了并且告警出来了，平台应该从什么样的操作进行处理防止服务猛扩?


