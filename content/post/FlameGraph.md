---
title: "Java性能分析之火焰图"
date: 2014-07-16
lastmod: 2014-07-16
draft: false
tags: ["tech", "java", "tools", "performance", "tuning", "trace"]
categories: ["tech"]
description: "java性能调优时，弄明白具体的时间消耗分布非常关键，大部分的profiler工具都提供时间分布的树状图，看起来不够直观，这里为你介绍另外一个展现方式 - 火焰图，时间分布一目了然，当然问题解决起来也是事半功倍了。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# FlameGraph

火焰图[fn:1]，简单通过x轴横条宽度来度量时间指标，y轴代表线程栈的层次，简单明了，
容易找出具体的可优化点，非常方便，当然前提是我们通过profiler工具获取到profiler
数据。

## java profiler
java性能调优时，我们经常会用到profiler工具，但是很多时候你可能不知道，大部分的
profiler工具都是有问题的[fn:2][fn:3]，简单来说，profiler：增加开销；修改了你的
代码，导致java编译器的优化行为不确定；同时影响了代码的层次，层次越深自然也影响
执行效率。

当然如果你不是通过上面方式实现，而是通过获取on-cpu线程的线程栈方式，这又会带来
一个麻烦的问题：获取系统范围的线程栈，jvm必须处于safepoint[fn:4]状态，只有当线
程处于safepoint状态的时候，别的线程才能去获取它的线程栈，而这个safepoint是由jvm
控制的，这对于profiler非常不利，有可能一个很热的代码块，jvm不会在该代码块中间放
置safepoint，导致profiler无法获得该线程栈，导致错误的profiler结果。

上面的问题几个商用的profiler工具都存在，Oracle Solaris studio利用的是jvmti的一
个非标准接口AsyncGetCallTrace来实现，不存在上面问题，Jeremy Manson也利用该接口
实现了一个简单的profiler工具：[[https://code.google.com/p/lightweight-java-profiler/wiki/GettingStarted][Lightweight Asynchronous Sampling Profiler]]，我们
的火焰图的数据来源就是通过它来获取的。

## lightweight-java-profiler
当然，这个工具只支持hotspot的vm，需要你自己编译，有些问题需要注意：
 - 如果你需要在rhel上编译，需要安装4.6以上版本gcc[fn:5]，4.4版本不支持。
 - 如果你需要在ubunt上编译，可能会碰到编译错误[fn:6]。

编译的时候，需要主要修改BITS参数，如果你要编译64Bit，使用命令：
: make BITS=64 all

使用方法很简单，直接在你的启动命令上添加如下参数：
: -agentpath:path/to/liblagent.so[:file=name]

启动之后，会在启动目录下生成trace.txt文件（缺省），该文件就是我们需要的采样数据。

另外有几个参数可在编译时修改，都在global.h文件中。首先是采样的频率，缺省是100次
每秒；另外是最大采样的线程栈，缺省3000，超过3000就忽略（对于复杂的应用明显不够）
；最后是栈的深度，缺省是128（对于调用层次深的应用调大）。当然你记录的东西越多，
也会有性能损耗，我调成30000+256，一刻钟生成200M文件。

另外特别需要注意，trace不是实时写入，而是在应用shutdown的时候才写入的，别kill应
用，否则trace里面什么都没有。

## 生成火焰图
大神Brendan Gregg[fn:7]已经帮你做好了，直接check大神的项目：
``` shell
git clone http://github.com/brendangregg/FlameGraph
cd FlameGraph
./stackcollapse-ljp.awk < ../traces.txt | ./flamegraph.pl > ../traces.svg
```

效果图截屏参考：
[[img:flamegraph.png][火焰图]]

真正火焰图指上去会显示栈信息。

看火焰图，找到那些长条分析，基本上就有方向了。


# Footnotes

[fn:1] [[http://www.brendangregg.com/flamegraphs.html][Flame Graphs]]

[fn:2] [[http://pl.cs.colorado.edu/papers/profilers-pldi10.html][T Mytkowicz, A Diwan, M Hauswirth, PF Sweeney. "Evaluating the Accuracy of Java Profilers,"]]

[fn:3] [[http://jeremymanson.blogspot.jp/2010/07/why-many-profilers-have-serious.html][Why Many Profilers have Serious Problems ]]

[fn:4] [[http://chriskirk.blogspot.jp/2013/09/what-is-java-safepoint.html][What is java safepoint]]

[fn:5] [[http://superuser.com/questions/381160/how-to-install-gcc-4-7-x-4-8-x-on-centos][How to Install gcc 4.7.x/4.8.x on CentOS]]

[fn:6] [[https://code.google.com/p/lightweight-java-profiler/issues/detail?id%3D3][build error on Ubuntu 13.10 (Saucy)]]

[fn:7] [[http://brendan.gregg.usesthis.com/][Brendan Gregg]]
