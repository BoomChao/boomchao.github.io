---
date : '2025-03-23T15:12:50+08:00'
draft : true
title : '分布式系统设计与实践'
tags : ["分布式"]
categories: ["分布式"]
---



## 第六章:分布式服务调用中间件
RPC(Remote Procedure Call) 远程过程调用
RPC的基础是序列化和反序列化，因为一切RPC消息，参数、返回值和异常等都需要被序列化和反序列化之后才能跨节点传递；
业界对其也有非常成熟的方案，比如JSON、XML、Java Object Serialization等序列化方式，但是上面提到的技术各有其
缺点，比如JSON、XML是基于文本的，编码效率低，Java Objet Serialization只限于Java语言
为此，为了解决上面的一些通用的问题，Google 开源的 PB(Protocol Buffers) 则很好的解决了这个问题
在PB的基础上，Google 开源除了它的RPC解决方案 gRPC，gRPC的传输使用的是HTTP-2协议，采用HTTP的好处能够支持各种
平台(包括移动端设备)以及各种语言。采用HTTP-2是为了规避前面提到的HTTP-1的一些缺陷，比如队头阻塞和不支持服务器推送等问题

