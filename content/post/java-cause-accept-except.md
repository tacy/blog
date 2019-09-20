---
title: "Jetty接入异常"
date: 2019-09-19
lastmod: 2019-09-19
draft: false
tags: ["tech", "java", "nio", "jettp"]
categories: ["tech"]
description: "集成平台，基于Jetty，暴露的服务调用无响应，端口监听正常（通过telnet验证），类似平台挂起"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# 故障现象
客户现场测试，部署的一个集成平台，Java，基于RHEL，在自己机器上测试一切正常，部署到客户机器怎么都无法调通，后台也没有任何日志。

# 分析解决
现场反馈是在k8s容器里面部署不行，后面了解到的情况是即使直接部署在虚拟机上也是无法调通，也就可以排除容器网络的问题。

首先怀疑是网络问题，同事的反馈显示，端口是能正常telnet的，curl也能看到连接正常：
![jetty curl](/img/jetty2.jpg)

上图能看到连接已经建立了，但是server端迟迟没有返回。由于这里已经是直接本地调用了，查看了一下iptables，没发现有rule会影响29090端口的服务调用，自然怀疑是集成平台本身出了问题。

接着就查看了一下集成平台的java线程栈，没有发现有线程挂住的情况，里面的线程都是空的，结合日志情况看，服务是没有被调用的。

用抓包`tcpdump -i lo tcp -w esb.pcap`分析了一下，确认请求是已经收到了：
![jetty wireshark](/img/jetty3.jpg)

也就是说从tcp协议层，包已经被接收了，为什么会出现这样的情况呢？按理来说，接收到包，中断，然后应用介入，acceptor读取请求，但是从线程栈看，没发现有什么异常情况。

用`ss -tni -o "dport = :29090`确认一下
![jetty ss](/img/jetty4.jpg)

从输出看，recv-q里面有88字节待读取。结合抓包截图，看到我们的请求大小就是88字节，也就是说，协议层面没有问题，包已经送达了，也没有overflow drop问题，问题确认出现在java层面，怀疑nio问题，我的最初怀疑是jdk，最后通过降低jetty版本解决，时间有限，没有再做更多分析。
