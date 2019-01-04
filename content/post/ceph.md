---
title: "ceph notes"
date: 2015-04-08
lastmod: 2015-04-08
draft: false
tags: ["tech", "storage", "virtualization", "ceph"]
categories: ["tech"]
description: "ceph笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# ceph重要的概念
ceph底层是RADOS(A reliable, autonomous, distributed object storage), RADOS由OSD和MON组成, 是ceph核心.
  - OSD: Object Storage Device
  - MON: 维护整个ceph集群的全局状态(cluster map)

另一个核心概念是CRUSH, ceph没有中心控制节点, 通过CRUSH算法结合MON维护的cluster map, 能够计算出任意对象的存储位置.


Object
Pool
Placement Group
OSD
MON

# CRUSH
CRUSH是用来确定数据存储位置的算法, client利用CRUSH获得了直接和OSD交互的能力, ceph通过CRUSH实现了去中心化的架构, 避免了中心节点带来的SPOF, 性能瓶颈和物理限制, 实现了理论上的无限扩展能力.




# PG
每个Pool都有自己的PG个数定义, 在Pool创建之后, PG的个数就无法更改.

首先需要了解的是, PG越多, 数据越分散, 推荐的计算方法:

(N * 10) / R

N代表OSD的个数, R代表副本数()

PG状态

# 实施建议

  - 单个Host上的OSD不能太多,理论上你可以挂载到硬件上限, 但是需要考虑到当Host失败的时候, 会带来巨大的迁移流量和长的恢复时间.

# client
当client需要写入文件到cluster中时, 首先从MON获取cluster map副本, 然后根据CRUSH算法获得PG id, 找到PG所属OSD set的Primary OSD(第一个), 之后client往该OSD写入数据, 副本由Primary OSD负责写入. 该写入操作完成需要等所有数据落盘(好像是写入journal就算完成?), 然后Primary OSD 返回ACK到client, 完成写入操作.

当然其实前面还有事情要做, 先要把文件按照object大小切分成多个object, 然后每个object会存入到不同的PG.


# ceph性能相关

## 写相关

### 一次写入产生的数据量
1M数据写入3副本集群, 会产生6.4M左右的数据, 其中包括:
  - journal三份
  - data三份(含副本)
  - PG log
总计下来大概是6.4M左右
