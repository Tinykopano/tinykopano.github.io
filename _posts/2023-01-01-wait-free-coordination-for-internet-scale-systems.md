---
title: ZooKeeper: Wait-free coordination for Internet-scale systems
author: kopano
date: 2023-01-01 14:22:00 +0800
categories: [Course,Paper]
tags: [distributed system, zookeeper]
---

# ZooKeeper: 互联网级系统的无等待协调

## 摘要

在本文中，我们描述了ZooKeeper，一个用于协调分布式应用程序进程的服务。由于ZooKeeper是关键基础设施的一部分，ZooKeeper的目的是为在客户端建立更复杂的协调基元提供一个简单和高性能的内核。它在一个复制的集中式服务中整合了群组消息传递、共享寄存器和分布式锁服务的元素。ZooKeeper提供的接口具有共享寄存器的免等待功能，以及类似于分布式文件系统的缓存失效的事件驱动机制，以提供一个简单而强大的协调服务。

ZooKeeper接口能够实现高性能的服务。除了无等待的特性外，ZooKeeper还为每个客户端提供了以下保证：请求的FIFO执行和所有改变ZooKeeper状态请求的可线性化。这些设计决定使高性能的处理管道得以实现，读取请求由本地服务器满足读取请求。我们展示了对于目标工作负载，即2:1到100:1的读写比，ZooKeeper可以每秒处理几万到几十万的事务。这种性能使ZooKeeper可以被客户端应用程序广泛使用。

## 一、   引言

## 二、   Zookeeper服务

## 三、   Zookeeper应用

## 四、   Zookeeper实现

## 五、   评估

## 六、   相关工作

## 七、   结论

