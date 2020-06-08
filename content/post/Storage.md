---
title: "Storage notes"

date: 2014-03-21
lastmod: 2014-03-21
draft: false
tags: ["tech", "storage", "performance"]
categories: ["tech"]
description: "关于存储性能的一些相关知识整理，主要包括性能方面。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# 硬盘IOPS[fn:3][fn:4][^1]

机械硬盘IO时间组成：
Rotational Delay: 盘旋转时间(盘面旋转半圈的时间，RPM/60/2)
Seek time: 寻道时间，产品参数
Transfer delay: 读取和传输时间 (读取8K数据为例, 6Gbps的盘, 4ms seek time, 2ms Rota latency, (1000ms-4ms-2ms)/((6Gbps/8)*1024*1024/8). )

| size |   RPM | Seek time | Avg Rota latency | Throught | Random IOPS      |
|      |       |           | 0.5/(RPM/60)     |          | 1/(seek+latency) |
|------+-------+-----------+------------------+----------+------------------|
|  2.5 | 15000 | 4ms       | 2ms              | 6Gbps    | ~175-210         |
|  2.5 | 10000 | 4ms       | 3ms              | 6Gbps    | ~175-210         |
|  3.5 | 15000 | 8ms       | 2ms              | 6Gbps    | ~140             |
|  3.5 | 10000 | 8ms       | 3ms              | 6Gbps    | ~140             |
|  2.5 |  7200 | 10ms      | 4ms              | 3Gbps    | ~75-100          |
|      |       |           |                  |          |                  |

# RAID对IOPS影响[fn:1][fn:2][fn:6]

| RAID Level | Read | write |
|------------+------+-------|
|          0 | 1    | 1     |
|     1 / 10 | 1    | 2     |
|          5 | 1    | 4     |
|          6 | 1    | 6     |
参照上表，一个raid6的存储，一个写请求需要消耗6个IOPS，简单计算一下，如果你需要
一个250IOPS的存储，一半读一半写，那就意味着你需要准备(250*50% + 250*50%*6)=875
的IOPS存储。

# Latency comparison[fn:5][^2]

``` shell
L1 cache reference                            0.5 ns
Branch mispredict                             5   ns
L2 cache reference                            7   ns             14x L1 cache
Mutex lock/unlock                            25   ns
Main memory reference                       100   ns             20x L2 cache, 200x L1 cache
Compress 1K bytes with Zippy              3,000   ns
Send 1K bytes over 1 Gbps network        10,000   ns    0.01 ms
Read 4K randomly from SSD*              150,000   ns    0.15 ms
Read 1 MB sequentially from memory      250,000   ns    0.25 ms
Round trip within same datacenter       500,000   ns    0.5  ms
Read 1 MB sequentially from SSD*      1,000,000   ns    1    ms  4X memory
Disk seek                            10,000,000   ns   10    ms  20x datacenter roundtrip
Read 1 MB sequentially from disk     20,000,000   ns   20    ms  80x memory, 20X SSD
Send packet CA->Netherlands->CA     150,000,000   ns  150    ms
```

# Footnotes

[fn:1] [[http://www.techrepublic.com/blog/the-enterprise-cloud/calculate-iops-in-a-storage-array/][Calculate IOPS in a storage array]]

[fn:2] [[http://www.yellow-bricks.com/2009/12/23/iops/][IOPS]]

[fn:3] [[http://en.wikipedia.org/wiki/IOPS][Wiki IOPS]]

[fn:4] [[http://www.csee.umbc.edu/~olano/611s06/storage-io.pdf][Storage IO]]

[fn:5] [[https://gist.github.com/jboner/2841832][Latency Numbers Every Programmer Should Know]]

[fn:6] [[http://www.wmarow.com/strcalc/][IOPS Calculator]]

[^1]: [Storage performance: IOPS, latency and throughput](http://rickardnobel.se/storage-performance-iops-latency-throughput/)

[^2]: [Latency Numbers Every Programmer Should Know](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html)
