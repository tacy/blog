---
title: "java应用未关闭的http连接分析"
date: 2019-07-18
lastmod: 2019-07-18
draft: false
tags: ["tech", "java", "tcpip", "network"]
categories: ["tech"]
description: "描述在java应用中，如果创建的http连接没有主动关闭，而是直接将关联对象置未空，引发的现象分析"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
正常情况下，我们创建的http连接都需要主动关闭（也有很多http连接池管理的实现帮助你，你也需要release回到池中）。但依然有些丑陋的代码，忘记主动关闭，这些没有关闭的连接最终都会被回收，用不同的方式。

存在两种方式：

一种是连接发起端垃圾回收。由于http对象没有任何引用，gc会把它回收掉，这种情况下，连接不是采用四次挥手的正常端口模式，而是通过发送RESET包断开，没有处理的数据也会直接扔掉。

一种是服务端断开。由于keepalive存在，一般服务端都会有keepalive超时设置。如果到了超时时间，客户端没有断开连接，服务端会主动断开。这种情况下，服务端会采用四次挥手方式断开连接，但是由于客户端对象已经没有任何引用，不会调用close方法，客户端虽然接收到了服务端发过来的fin包，但是不会回fin包，所以连接无法正常断开：客户端将处于close-wait状态，等待gc回收，服务端将处于fin-wait-2状态，并等待超时，进入time-wait状态。

如果你在系统中看到大量的reset包，或者看到tcp连接处于close-wait/fin-wait-2状态，一般情况下都是连接没有正常关闭导致。
