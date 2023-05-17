---
title: 'Object Storage on CRAQ: High-throughput chain replication for read-mostly workloads'
date: 2023-05-07 19:32:00 +0800
categories: [Course,Paper]
tags: [distributed system]
---

# CRAQ上的对象存储：针对大部分为读的工作负载的高吞吐量链式复制

## 摘要

大规模的存储系统通常在许多可能出现问题的组件上复制和划分数据，以提供可靠性和可扩展性。然而，许多商业部署的系统，特别是那些为客户交互使用而设计的系统，为了追求更高的可用性和更高的效率，牺牲了更强的一致性性，以获得更强的可用性和更高的吞吐量。

本文描述了CRAQ的设计、实现和评估，这是一个分布式对象存储系统，挑战这种不灵活的权衡。我们的基本方法是对链式复制的改进，在保持强大的一致性的同时，极大地提高了读取吞吐量。通过在所有对象复制中分配负载，CRAQ随着链的大小线性扩展而不增加一致性协调。同时，当未提交操作对某些应用来说已经足够时，它还会暴露出较弱的一致性保证，这在系统高速运转的时期特别有用。本文探讨了跨多个数据中心的地理复制CRAQ存储的额外设计和实现考虑，以提供本地优化的操作。我们还讨论了多对象原子更新和大对象更新的组播优化。

<!-- more -->