---
title: "XBMC插件之网易公开课"
date: 2013-07-23
lastmod: 2013-07-23
draft: true
tags: ["tech", "python", "xmbc", "raspberrypi"]
categories: ["tech"]
description: "自己为XBMC写的一个网易公开课插件，满足课程分类/学校分类/查找功能"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# openjdk enable dtrace
在archlinux，先安装aur包systemtap-git，还需要cp源代码里面的includes/sys/*到/usr/includes/sys/目录下，然后编译jdk才能启用enable-dtrace


# jvm缺省参数
Default value :
```
 java -XX:+PrintFlagsFinal | grep ParallelGCThreads
 uint  ParallelGCThreads                        = 4
```

If you have a running process jinfo <processId>, if it's not present in the output, it is using the default value (look under VM Flags)


# keepalive
用tomcat说明

如果服务端需要和客户端保持长连接，需要满足两个条件：首先需要配置服务端connector的keepAliveTimeout和MaxKeepAliveRequests参数；其次需要客户端使用使用http1.1协议。满足这两个条件，tomcat会保持该连接不会断开，直到超过KeepAliveTimeout时间设置

``` shell
    <Connector port="8090" protocol="HTTP/1.1"
               connectionTimeout="20000" keepAliveTimeout="-1" maxKeepAliveRequests="-1"
               redirectPort="8443" />
```

下面是一个实际的截图信息：

``` shell
服务端连接状态：
[root@wfpprdap21 ~]# ss -tnio 'sport = :8090'
State      Recv-Q Send-Q                                              Local Address:Port                                                             Peer Address:Port
ESTAB      0      0                                              ::ffff:10.24.20.21:8090                                                        ::ffff:10.24.20.1:10132
         ts sack cubic wscale:7,7 rto:205 mss:1448 cwnd:10 ssthresh:16 lastsnd:1177991 lastrcv:113022304 lastack:1177991 rcv_space:28960

客户端发起连接：
[root@wfpprdap01 config]# ncat 10.24.20.21 8090

```

但是有一种情况需要注意，如果网络情况复杂，例如客户端和服务端中间有防火墙存在，防火墙默认会有session超时时间，如果超时，会直接清除session，并不会通知连接双方，这种情况下，连接双方会出现连接断开异常（客户端和服务端都不知道连接已经被断开了，双方的包会被防火墙drop掉），为了防止上面的情况发生，我们需要借助tcp协议层的keepalive机制。

在linux中，tcp连接默认实现有keepalive timer机制，当连接空闲超过一定时间，会触发keepalive prober，默认情况下linux的keepalive probe触发条件是7200s，也就是说连接只有空闲了7200s，才会触发keepalive probe，而这个时间肯定是要超过防火墙的session的超时时间。你需要缩短这个触发时间（只需要调整linux tcp参数）

但是，应用需要启用keepalive timer，需要在你的监听socket上设置了so_keepalive属性。例如tomcat，nio和bio只有设置了socket.soKeepAlive才行

用系统工具ss/netstat监控到连接上启用了keepalive timer。

``` shell
    <Connector port="8090" protocol="org.apache.coyote.http11.Http11NioProtocol"
               connectionTimeout="20000" keepAliveTimeout="-1" maxKeepAliveRequests="-1" socket.soKeepAlive="true"
               redirectPort="8443" />
```

下面是一个实际的截图信息：

``` shell
服务端连接状态：
[root@wfpprdap21 ~]# ss -tnio 'sport = :8090'
State      Recv-Q Send-Q                                              Local Address:Port                                                             Peer Address:Port
ESTAB      0      0                                              ::ffff:10.24.20.21:8090                                                        ::ffff:10.24.20.1:5718                timer:(keepalive,51sec,0)
         ts sack cubic wscale:7,7 rto:203 mss:1448 cwnd:10 ssthresh:16 lastsnd:8441 lastrcv:114191918 lastack:8441 rcv_space:28960

客户端发起连接：
[root@wfpprdap01 config]# ncat 10.24.20.21 8090

```

另外，如果你的应用属于商业产品，并且不支持设置so_keepalive，你可以尝试使用[libdontdie](https://github.com/flonatel/libdontdie)。


# socket so_linger[^1]
socket close支持两种方式：一种是正常关闭（四次挥手，等待数据都发送完毕），一种是直接中断关闭（直接发送RESET，不会等数据发送完）。

一般情况下，服务端不会主动关闭连接，但有些情况，客户端写的有问题（例如不关闭连接），这时候，服务端一般会有超时机制，主动关闭连接。这类情况出现在高并发系统上，会带来问题，我们知道，主动关闭连接的一方，连接关闭之后，会处于time-wait（linux是60s），如果系统的并发超过一分钟6w，会出现端口耗尽的问题。

碰到这种情况，可以利用socket的so_linger特性，启用socket的so_linger，并且设置超时时间为0，服务端就会采用直接发送RESET包的方式关闭连接，保持端口可用

[^1]: [TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)
