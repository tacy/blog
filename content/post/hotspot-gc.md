---
title: "HotSpot GC日志解读"
date: 2014-03-24
lastmod: 2014-03-24
draft: false
tags: ["tech", "java", "tuning", "performance"]
categories: ["tech"]
description: "对于java应用的优化，GC是重要的一环，而读懂GC日志是调优GC的关键，这里尝试对GC日志做一些解读。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# HotSpot GC
对于Java应用的优化，GC是重要的一环，而读懂GC日志是调优GC的关键，本文尝试对GC
做一些解读，测试使用的JVM版本是1.6.0_45-b06（64Bit）。

## 几个概念
### Serial & Parallel
Serial收集意味着单线程操作，而Parallel是尽可能的并行，在多CPU机器上，切分收集
任务到多个CPU运行，能够更快的完成收集操作，相对更复杂。

### Concurrent & Stop-the-world
STW收集策略指在GC期间，应用挂起，完成GC操作之后，应用继续运行，Concurrent策略
是指GC操作和应用并行，只是在做一些必要操作的时候挂起应用。STP相比Concurrent简
单，因为STW冻结了VM Heap，收集期间对象不会改变，能更快的完成GC操作，缺点是带来
比较长时间的应用挂起，有些应用无法接受；而Concurrent在收集期间应用同时运行，会
带来VM Heap内对象的变化，设计起来更复杂，需要更大的VM Heap来完成，调优起来也更
复杂。

### Compacting & Non-compacting & Copying
Compacting是指GC之后，移动所有的活着的对象到一起，形成连续的空闲空间，能够高效
的完成后续的对象分配任务，只需简单的移动指针。Non-compacting则是就地释放对象，
也不会移动活着的对象，相对Compacting能更快完成，缺点是分配对象复杂，需要管理
碎片空间。Copying是指拷贝存活对象到另外的内存空间，清空源内存空间，后续对象分配
更容易，缺点是拷贝对象的开销。

## HotSpot JVM组成
包括两块，一块是JVM heap，另一块是Permanent Generation，主要我们关注JVM heap，
JVM heap又分为两块：Young Generation和Old Generation（Tenured）。新分配对象和
短生命周期对象一般都在Young Generation（大对象会直接分配到Old Generation中）,
Old Generation主要是长生命周期的对象。

JVM heap大小通过-Xms和-Xmx控制，一般生产环境会设置相同的值，如果不是设置相同
大小，每一次JVM调整heap时候都会导致Full GC，对系统运行产生一定影响。

Permanent Generation大小通过-XX:PermSize和-XX：MaxPermSize控制，大小主要看
项目jar包的多少，这里主要存放的是class元数据和VM自己运行的数据，设置大小一致
能避免Permanent空间调整导致的Full GC。

Young Generation分为三块，一块是Eden space，另外两块是大小相等的Survivor space，
我们这里简称S0、S1。Young Generation大小的控制有下面参数：
-XX:NewSize / -XX:MaxNewSize (最大最小）
-XX:NewRatio=3 指定Young Generation和Old Generation的比例是1：3
-Xmn 最大最小保持一致
JVM也提供了自适应的调整策略，如果你没有明确指定这些值，JVM会自己调整大小（搜索
HotSport VM Adaptive）。

Eden space和Survivor space大小通过-XX:SurvivorRatio=6参数可以控制，等于6表示
Survivor space大小是Eden Space的1/6，是整个Young Generation的1/8（注意是两个
Survivor space）。一般这个参数不会设置，JVM会自动计算最优比例，如果设置不合理
容易导致Survivor space overflow（过小）或者Survivor space浪费(过大），Overflow
的后果会导致放不下的对象直接迁移到Old Generation，带来不必要的Full GC。具体的
行为可以通过-XX:+PrintTenuringDistribution观察。

## GC的策略
记住几点：
- GC的回收策略有不同的作用区域；
- 所有的Serial和Parallel策略都是STW的；
- 所有对Young Gen的回收都是Copying策略；
- Concurrent策略都是非STW的；

作用域和参数参考下表[fn:5]：

| Young Collector   | Old Collector               | JVM Option             |
|-------------------+-----------------------------+------------------------|
| Serial(DefNew)    | Serial Mark-Sweep-Compact   | -XX:+UseSerialGC       |
|-------------------+-----------------------------+------------------------|
| Parallel scavenge | Serial Mark-Sweep-Compact   | -XX:+UseParallelGC     |
| (PSYoungGen)      | (PSOldGen)                  |                        |
|-------------------+-----------------------------+------------------------|
| Parallel scavenge | Parallel Mark-Sweep-Compact | -XX:+UseParallelOldGC  |
| (PSYoungGen)      | (ParOldGen)                 |                        |
|-------------------+-----------------------------+------------------------|
| Parallel(ParNew)  | Serial Mark-Sweep-Compact   | -XX:+UseParNewGC       |
|-------------------+-----------------------------+------------------------|
| Serial(DefNew)    | Concurrent Mark Sweep       | -XX:UseConcMarkSweepGC |
|-------------------+-----------------------------+------------------------|
| Parallel(ParNew)  | Concurrent Mark Sweep       | -XX:UseConcMarkSweepGC |
|                   |                             | -XX:+UseParNewGC       |
|-------------------+-----------------------------+------------------------|
| G1                | G1                          | -XX:+UseG1             |

### SerialGC
YoungGen和OldGen回收策略，GC使用单线程，回收期间STP，适用于单处理器服务器和小的
堆（一两百兆）。
``` shell
10.590: [GC 10.590: [DefNew: 34944K->3140K(39296K), 0.0171370 secs] 95253K->63449K(126720K), 0.0171930 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
10.787: [GC 10.787: [DefNew: 38084K->1913K(39296K), 0.0242650 secs] 98393K->65358K(126720K), 0.0243320 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
10.842: [GC 10.842: [DefNew: 36857K->3477K(39296K), 0.0134890 secs] 100302K->66922K(126720K), 0.0136120 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
11.012: [Full GC 11.012: [Tenured: 63445K->67278K(87424K), 0.2908000 secs] 82061K->67278K(126720K), [Perm : 64767K->64760K(64768K)], 0.2908740 secs] [Times: user=0.29 sys=0.00, real=0.29 secs]
11.954: [Full GC 11.954: [Tenured: 67278K->69894K(87424K), 0.2313190 secs] 91392K->69894K(126720K), [Perm : 70207K->70207K(70208K)], 0.2313890 secs] [Times: user=0.24 sys=0.00, real=0.23 secs]
```
日志中的DefNew表示Serial回收，OldGen回收也是采用Serial Mark-Sweep-Compact。从日
志里面能获取到很多信息：
看第一行，34944K表示回收之前YoungGen对象大小，3140K表示回收之后存活的对象大小(结
合后续MinorGC小节里面的TargetSurvivorRatio可以准确计算出Survivor space大小），耗
时0.017930S，95253K表示回收之前JVM Heap对象占用空间大小，63449K表示回收之后堆内对
象大小，126720K表示整个堆大小。（126720K-39296K）=87424K是Old Gen大小，被回收的
垃圾对象为（95253K-63449K）=31804K，移动到OldGen对象（34944K-3140K-31804K）=0K，
回收所用的User时间为0.02S，系统调用时间0S，时间周期0.02S，三个值区别参考[fn:10]。
看第四行，你会发现Old Gen还有空间，触发GC的是Perm Gen，已经满了，尝试回收，发现
回收不了64767K -> 64760K，扩容Perm Gen，触发Full GC。

### Parallel Scavenge
Young Gen回收策略，并发回收，回收期间STW。
``` shell
13.782: [GC [PSYoungGen: 18768K->3368K(29632K)] 93584K->80361K(117056K), 0.0082680 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
14.015: [GC [PSYoungGen: 19944K->5449K(31424K)] 96937K->85551K(118848K), 0.0097840 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
14.024: [Full GC [PSYoungGen: 5449K->0K(31424K)] [PSOldGen: 80101K->85415K(87424K)] 85551K->85415K(118848K) [PSPermGen: 76528K->76528K(153344K)], 0.2642610 secs] [Times: user=0.27 sys=0.00, real=0.27 secs]
```
日志中的PSYoungGen表示Parallel Scavenge，并行对YoungGen回收，PSOldGen表示Serial
Mark-Sweep-Compact，单线程对OldGen进行回收。这里Minor GC中的user时间大于real时
间是由于多线程并行导致。

### Parallel Mark-Sweep-Compact
Old Gen回收策略，并发回收，回收期间STW。
``` shell
14.488: [GC [PSYoungGen: 19568K->7759K(29760K)] 96421K->87358K(117184K), 0.0180290 secs] [Times: user=0.06 sys=0.00, real=0.02 secs]
14.632: [GC [PSYoungGen: 25551K->288K(30720K)] 105150K->87492K(118144K), 0.0135020 secs] [Times: user=0.03 sys=0.00, real=0.02 secs]
14.646: [Full GC [PSYoungGen: 288K->0K(30720K)] [ParOldGen: 87204K->66401K(87424K)] 87492K->66401K(118144K) [PSPermGen: 76492K->76370K(153408K)], 0.8717190 secs] [Times: user=2.76 sys=0.00, real=0.87 secs]
```
日志中的ParOldGen表示Paralle Mark-Sweep-Compact，并行对OldGen进行回收。

两个Parallel策略也被称为Throught回收策略，由于并行进行GC操作，提高GC效率，获取
更高的应用吞吐量，但是Parallel同时会导致比较长的STW，对延迟敏感的应用不适用。

### Concurrent-Mark-Sweep
Old Gen回收策略，也被称为低延迟的GC策略，很少的STP操作，但是需要Old Gen保持一定
的空闲空间来防止碎片问题（没有Compact，YoungGen promotion进来的对象需要连续的空
间，容易导致fragmentation）。它分为六个步骤执行[fn:6]：
- Initial Mark (STW)
  标记root object直接引用的对象
- Concurrent Mark
  应用线程继续响应，标记第一步标记对象引用的所有对象。
- Concurrent Preclean
- Remark (STW)
- Concurrent Sweep
- Concurrent Reset
打开CMS同时缺省打开ParNewGC策略对YoungGen进行并行回收。
下面日志是一个完整的CMS周期，详细解释请参考[fn:7]，注意这里的日志是JDK8的，
主要是大牛写的很详细，我就懒得自己整理了。

``` shell
1.[GC [1 CMS-initial-mark: 463236K(515960K)] 464178K(522488K), 0.0018216 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
2.[GC[ParNew: 6528K->702K(6528K), 0.0130227 secs] 469764K->465500K(522488K), 0.0130578 secs] [Times: user=0.05 sys=0.00,real=0.01 secs]
3.[GC[ParNew: 6526K->702K(6528K), 0.0136447 secs] 471324K->467077K(522488K), 0.0136804 secs] [Times: user=0.04 sys=0.01,real=0.01 secs]
4.[GC[ParNew: 6526K->702K(6528K), 0.0161873 secs] 472901K->468830K(522488K), 0.0162411 secs] [Times: user=0.05 sys=0.00,real=0.02 secs]
5.[GC[ParNew: 6526K->702K(6528K), 0.0152107 secs] 474654K->470569K(522488K), 0.0152543 secs] [Times: user=0.05 sys=0.00,real=0.02 secs]
...
6.[GC[ParNew: 6526K->702K(6528K), 0.0144212 secs] 481073K->476809K(522488K), 0.0144719 secs] [Times: user=0.05 sys=0.00,real=0.01 secs]
7.[CMS-concurrent-mark: 1.039/1.154 secs] [Times: user=2.32 sys=0.02, real=1.15 secs]
8.[CMS-concurrent-preclean: 0.006/0.007 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
9.[GC[ParNew: 6526K->702K(6528K), 0.0141896 secs] 482633K->478368K(522488K), 0.0142292 secs] [Times: user=0.04 sys=0.00,real=0.01 secs]
10.[GC[ParNew: 6526K->702K(6528K), 0.0162142 secs] 484192K->480082K(522488K), 0.0162509 secs] [Times: user=0.05 sys=0.00,real=0.02 secs]
11.[CMS-concurrent-abortable-preclean: 0.022/0.175 secs] [Times: user=0.36 sys=0.00, real=0.17 secs]
12.[GC[YG occupancy: 820 K (6528 K)][Rescan (parallel) , 0.0024157 secs][weak refs processing, 0.0000143 secs][scrub string table, 0.0000258 secs] [1 CMS-remark: 479379K(515960K)] 480200K(522488K), 0.0025249 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
13.[GC[ParNew: 6526K->702K(6528K), 0.0133250 secs] 441217K->437145K(522488K), 0.0133739 secs] [Times: user=0.04 sys=0.00, real=0.01 secs]
14.[GC[ParNew: 6526K->702K(6528K), 0.0125530 secs] 407061K->402841K(522488K), 0.0125880 secs] [Times: user=0.04 sys=0.00,real=0.01 secs]
...
15.[GC[ParNew: 6526K->702K(6528K), 0.0121435 secs] 330503K->326239K(522488K), 0.0121996 secs] [Times: user=0.04 sys=0.00,real=0.01 secs]
16.[CMS-concurrent-sweep: 0.756/0.833 secs] [Times: user=1.68 sys=0.01, real=0.83 secs]
17.[CMS-concurrent-reset: 0.009/0.009 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
```

这里时间统计多说一句，比如倒数第二行中的0.756/0.833 secs，0.756表示运行sweep
时间，0.833表示时钟运行的时间（等于real），由于sweep操作是并行的，所以user大
于real。

CMS周期期间，因为Minor GC会继续运行（Minor GC进入会挂起CMS的concurrent操作），
会出现Minor GC promotion failure的情况（YoungOld中的长生命周期对象需要移动到
Old Gen中，但是Old Gen空间不足或者没有连续的空间块来容纳移动的对象），这会导
致CMS中断，同时触发Full GC操作，参考下面日志：

``` shell
64.528: [GC 64.528: [ParNew: 19128K->2112K(19136K), 0.0088010 secs] 108133K->94575K(128960K), 0.0088840 secs] [Times: user=0.02 sys=0.01, real=0.01 secs]
64.538: [GC [1 CMS-initial-mark: 92463K(109824K)] 94693K(128960K), 0.0057310 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
64.544: [CMS-concurrent-mark-start]
64.732: [GC 64.732: [ParNew: 19122K->2112K(19136K), 0.0158700 secs] 111585K->98845K(128960K), 0.0159520 secs] [Times: user=0.06 sys=0.00, real=0.02 secs]
64.874: [CMS-concurrent-mark: 0.312/0.330 secs] [Times: user=1.17 sys=0.02, real=0.34 secs]
64.874: [CMS-concurrent-preclean-start]
64.886: [CMS-concurrent-preclean: 0.012/0.012 secs] [Times: user=0.05 sys=0.00, real=0.01 secs]
64.886: [CMS-concurrent-abortable-preclean-start]
64.933: [GC 64.933: [ParNew: 19136K->2112K(19136K), 0.0106420 secs] 115869K->102790K(128960K), 0.0107100 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
65.060: [GC 65.060: [ParNew: 19136K->2112K(19136K), 0.0145130 secs] 119814K->105946K(128960K), 0.0146300 secs] [Times: user=0.04 sys=0.00, real=0.02 secs]
65.156: [CMS-concurrent-abortable-preclean: 0.151/0.269 secs] [Times: user=0.89 sys=0.01, real=0.27 secs]
65.156: [GC[YG occupancy: 11152 K (19136 K)]65.156: [Rescan (parallel) , 0.0068730 secs]65.163: [weak refs processing, 0.0000630 secs] [1 CMS-remark: 103834K(109824K)] 114987K(128960K), 0.0070350 secs] [Times: user=0.02 sys=0.00, real=0.00 secs]
65.163: [CMS-concurrent-sweep-start]
65.250: [GC 65.250: [ParNew (promotion failed): 19136K->19136K(19136K), 0.8336480 secs]66.084: [CMS66.109: [CMS-concurrent-sweep: 0.112/0.946 secs] [Times: user=1.50 sys=0.22, real=0.95 secs]
 (concurrent mode failure): 107755K->92385K(109824K), 0.4802380 secs] 121588K->92385K(128960K), [CMS Perm : 80068K->78975K(133032K)], 1.3140100 secs] [Times: user=1.61 sys=0.22, real=1.31 secs]
66.816: [GC 66.816: [ParNew: 17024K->2112K(19136K), 0.0097300 secs] 109409K->95636K(128960K), 0.0098010 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
```

在CMS周期内，Minor GC发现无法promotion对象，导致concurrent mode failure，中断CMS
同时触发Full GC。注意所有的Full GC都是STW（CMS用的好像是SerialMarkSweepCompact）
。造成这种情况需要分析是什么原因导致的，如果是碎片，最好是增加OldGen大小，如果是
由于对象移动到OldGen太快，可以提前触发CMS，CMS触发通过OccupancyFraction控制，该
值JVM会动态调整，你可以通过观察日志计算它，比如下面日志中：

: 29.542: [GC [1 CMS-initial-mark: 56351K(109824K)] 58400K(128960K), 0.0031880 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]

该值在51左右（56351/109824=0.51），如果需要提前触发CMS，通过下面两个参数：
CMSInitiatingOccupancyFraction和UseCMSInitiatingOccupancyOnly（需要两个都设置，
否则只是第一次生效，后续JVM还是会自己调整）。另外就是避免对象移动过快，通过调优
Young Gen参数实现，当然治标的办法自然是减少对象分配速度。当然你也能增加OldGen。

下面的例子不在CMS周期之内，Minor GC发现无法promotin，触发Full GC，解决办法和
前面一样：

: 62.231: [GC 62.231: [ParNew (promotion failed): 19137K->19136K(19136K), 0.0403870 secs]62.271: [CMS: 99459K->77486K(109824K), 0.4331580 secs] 116817K->77486K(128960K), [CMS Perm : 80178K->79817K(132164K)], 0.4737430 secs] [Times: user=0.50 sys=0.01, real=0.47 secs]

另外一个触发Full GC的是PermGen，CMS缺省不会收集PermGen，容易导致Perm空间满，你
可以设置参数CMSClassUnloadingEnabled，让CMS也对PermGen进行收集：

还有一个参数‑XX:+ExplicitGCInvokesConcurrent，该参数可以让System.gc()不会触发
Full GC，这个比较有用。貌似-XX:+DisableExplicitGC会带来DirectByteMemory OOM问
题，如果你的程序中大量用了NIO。

## GC
GC分为Minor GC、Full GC（Major GC）。

### Minor GC
主要对Young Generation进行回收，Minor GC之后，eden space和一块survivor
space清空，一些生命周期长的对象会移动到Old Generation，另外的对象都会移动到另一
块Survivor space中（如果之前是S0清空，那么所有活着的对象都会被移动到S1，反之同
理），理解这一点对于分析GC log非常关键。

多长生命周期的对象会被移动[fn:4]，通俗点来说就是通过年龄控制。对象经历一次Minor GC增
加一岁。至于说多大年龄的对象会被移动，通过几个参数控制：TargetSurvivorRatio、
Survivor space size、MaxTenuringThreshold。每次Minor GC都会计算Survivor space
中期望保留的对象多少，通过（TargetSurvivorRatio * Survivor space size）计算得出
，具体的值可以通过打开-XX:+PrintTenuringDistribution参数查看日志中的Desired
survivor size。如果保留的对象大小超出期望的值，TenuringThreshold（对象年龄阀值
，超出会被迁移）就会降低，让更多的对象移动到Old Gen中，如果小于期望的值，当前值
不变。TenuringThreshold最大值通过MaxTenuringThreshold控制，范围是0～15。
TargetSurvivorRatio缺省是50，MaxTenuringThreshold缺省是15（ParalleGC,CMS是4）。

VM中的对象分配都在Eden中进行，由于每次Minor GC都会清空Eden，所以Eden中的
空间都是连续的（没有碎片），为了保证分配对象的高效[fn:1][fn:3]，VM使用了两个技术：
- bump-the-pointer
  由于是连续的空间，VM只需记住最晚分配对象的空间地址（在Eden顶上），下次分配的时
  候，只需变更地址就行。
- Thread-Local Allocation Buffers (TLAB)
  考虑到多线程的情况，如果多个线程去更新保存的地址，需要加锁，为了避免锁VM引入了
  TLAB技术，每个线程都会有一块自己的空间（TLAB）来分配对象（也采用BTP技术，TLAB
  位于Eden），这样就不需要引入锁。当一个TLAB分配满了之后（或者Minor GC来临），
  TLAB释放，GC完成之后，线程会在需要分配对象的时候申请TLAB。如果线程的TLAB满了需
  要继续申请对象，同时Eden还有空间的时候，那就要锁了。关于TLAB相关细节可以参考[fn:2]

考虑到有些Young Gen中的对象会被Old Gen引用，Minor GC也需要扫描Old Gen，未了避免
扫描整个Old Gen，VM采用Card table技术，尽量减少扫描的范围。VM会把Old Gen分隔为
512 bytes的块，card table是一个byte array，一个byte对应一个块，如果块的对象引用
到Young Gen对象，标示该card为dirty，这样Minor GC的时候只需要扫描dirty的table。

### Full GC
会对整个JVM Heap进行回收，包括YoungGen、OldGen、PermGen。主要是由Old Generation
空间不足触发（当然有很多种触发因素，比如system.gc()调用、Remote distributed GC
等），耗时会比较长，而且是STW操作（停止应用响应）。

### Concurrent GC
另外这里特别说明一下，Concurrent应该算是一个独立的GC类型，只对OldGen收集（可以包
括PernGen，目的就是尽量减少FullGC操作（对延迟敏感的应用无法忍受这么长的STP）。


## 几个相关参数
### GC日志
打印GC日志，建议在线上环境打开，开销很小，调优GC必不可少：
: -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -Xloggc:file
更多的信息：
: -XX:+PrintHeapAtGC -XX:+PrintTLAB

### VM参数查看
打印VM参数，可以通过几个命令，这里直接抄R大[fn:8]的内容，R大的blog绝对值得仔细
研究，也可以参考这篇blog[fn:9]，说明了输出的一些细节：
: -XX:+PrintCommandLineFlags
这个参数的作用是显示出VM初始化完毕后所有跟最初的默认值不同的参数及它们的值。这
个参数至少在Sun JDK 5上已经开始支持，Oracle/Sun JDK 6以及Oracle JDK 7上也可以使
用。Sun JDK 1.4.2还不支持这个参数。

: -XX:+PrintFlagsFinal
前一个参数只显示跟默认值不同的，而这个参数则可以显示所有可设置的参数及它们的值。
不过这个参数本身只从JDK 6 update 21开始才可以用，之前的Oracle/Sun JDK则用不了。
可以设置的参数默认是不包括diagnostic或experimental系的。要在-XX:+PrintFlagsFinal
的输出里看到这两种参数的信息，分别需要显式指定-XX:+UnlockDiagnosticVMOptions /
-XX:+UnlockExperimentalVMOptions 。

: -XX:+PrintFlagsInitial
这个参数显示在处理参数之前所有可设置的参数及它们的值，然后直接退出程序。“参数
处理”包括许多步骤，例如说检查参数之间是否有冲突，通过ergonomics调整某些参数的值
，之类的。结合-XX:+PrintFlagsInitial与-XX:+PrintFlagsFinal，对比两者的差异，就可
以知道ergonomics对哪些参数做了怎样的调整。


## 其他推荐阅读
### 一些值得阅读的blog，待补全
- [[https://blogs.oracle.com/jonthecollector/][Jon Masamitsu's Weblog]]
- [[http://rednaxelafx.iteye.com][Script Ahead, Code Behind]]


# Footnotes

[fn:1] [[http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/][Understanding Java Garbage Collection]]

[fn:3] [[http://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf][Memory Management in the Java Hotspot VM]]

[fn:4] Java performance by Charlie Hunt, Binu John

[fn:5] [[http://blog.ragozin.info/2011/09/hotspot-jvm-garbage-collection-options.html][HotSpot JVM garbage collection options cheat sheet]]

[fn:6] [[https://blogs.oracle.com/jonthecollector/entry/hey_joe_phases_of_cms][The Unspoken - Phases of CMS]]

[fn:7] [[https://blogs.oracle.com/jonthecollector/entry/the_unspoken_cms_and_printgcdetails][The Unspoken - CMS and PrintGCDetails]]

[fn:8] [[http://hllvm.group.iteye.com/group/topic/27945][JVM调优的"标准参数"的各种陷阱]]

[fn:2] [[https://blogs.oracle.com/jonthecollector/entry/the_real_thing][The real thing]]

[fn:9] [[https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-3-printing-all-xx-flags-and-their-values/][Useful JVM Flags – Part 3 (Printing all XX Flags and their Values)]]

[fn:10] [[http://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1][What do 'real', 'user' and 'sys' mean]]
