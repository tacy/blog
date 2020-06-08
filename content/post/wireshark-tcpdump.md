---
title: "wireshark和tcpdump相关笔记"
date: 2014-08-21
lastmod: 2014-08-21
draft: false
tags: ["tech", "network", "tools", "tcpip", "sniffer"]
categories: ["tech"]
description: "使用wireshark和tcpdump过程中的一些知识累积，两个都是解决网络问题的好工具，"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# tshark
tshark -V -O http -Y "(http.request and http.host contains ymatou)"  -T fields -e frame.number -e http.request.method -e http.request.full_uri -r ~/workspace/python/osx/lele/ymt-addproductv2.pcapng

## FILTER FIELD REFERENCE
tshark -G fields on the command line

# tcpump
1. Capture only HTTP GET and POST packets
`~$ sudo tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'`

I presume you pulled this from the "capture HTTP GET requests" filter example.

First, some notation:

var[n] means the nth byte of var.
var[n,c] means c bytes of var starting at offset n, e.g. var[3:4] would return bytes 3,4,5,6 of var.
& is the bitwise AND operation.
'>>' means bitwise shift right.
So, what do we have?

tcp[((tcp[12:1] & 0xf0) >> 2):4]
Let's deconstruct this into its individual parts:

tcp[12:1] takes 1 byte of the TCP segment (i.e. the packet including header) at offset 12. We can see from the structure that offset 12 (0xC) is the Data Offset field. Its definition is as follows:

Data offset (4 bits) specifies the size of the TCP header in 32-bit words. The minimum size header is 5 words and the maximum is 15 words thus giving the minimum size of 20 bytes and maximum of 60 bytes, allowing for up to 40 bytes of options in the header. This field gets its name from the fact that it is also the offset from the start of the TCP segment to the actual data.

Ok, cool. This field is only 4 bits, though, and tcp[12:1] takes a whole byte (8 bits). We only want the first four.

tcp[12:1] & 0xf0 takes that Data Offset byte and applies a bitwise AND operation, using the constant 0xF0. This is often known as a masking operation, since any bits in the constant set to 0 will also be set to zero in the output. It's easier to think about this in binary. 0xF0 is just 11110000, and since x & 0 for any bit x is always 0, and x & 1 for any bit x is always x, it follows that the zero bits in 0xF0 will "switch off" or mask the given bits, but leave the rest alone. In this case, if we imagine that tcp[12:1] is 10110101, the result would be 10110000 because the last four bits are masked to zero. The idea here is that, since the Data Offset field is only 4 bits long, but we have 8 bits, we mask out the irrelevant bits so we only have the first 4.

The problem here is that, numerically speaking, our 4 bits are in the "top" side of the byte. This means that instead of just having 1101 (our Data Offset bits), we have 11010000. If we just wanted to get the "raw" value of the Data Offset field, we'd right shift by four places. 10110000 >> 4 = 1101, i.e. we throw away the "bottom" 4 bits and shift the top four bits right. However, in this case you'll notice the filter only shifts across by 2 places, not 4.

This is where we want to refer back to the definition of the Data Offset field: it specifies the size of the header in 32-bit words, not bytes. So, if we want to know the length of the header in bytes, we need to multiply it by 4. As it turns out, a bitwise left-shift of 1 is the same as multiplying a number by 2, and a bitwise left-shift of 2 is the same as multiplying a number by 4.

Now, combine this with what we've already seen: >> 4 would make sense in the filter if you wanted to get that raw value of Data Offset, but then we want to multiply it by 4, which is equivilent to left shifting (<< 2), which cancels out part of that right shift, resulting in >> 2.

So, (tcp[12:1] & 0xf0) >> 2 extracts the Data Offset field and multiplies it by 4 to get us the size of the TCP header in bytes.

But wait, there's more!. In the filter you provided, we still need to do one more operation:

tcp[((tcp[12:1] & 0xf0) >> 2):4]
This is easier looked at if we used an intermediate variable:

$offset = ((tcp[12:1] & 0xf0) >> 2)
tcp[$offset:4]
What this does is get the first 4 bytes after the TCP header, i.e. the first 4 data bytes of the payload. In the filter you pulled this from, they were looking for HTTP GET requests using this filter:

port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420
The 0x47455420 constant is actually a numeric encoding of the ASCII bytes for GET (that last character is a space), where the ASCII values of those characters are 0x47, 0x45, 0x54, 0x20.

So, how does this work in full? It extracts the 4-bit Data Offset field from the TCP header, multiplies it by 4 to compute the size of the header in bytes (which is also the offset to the data), then extracts 4 bytes at this offset to get the first 4 bytes of the data, which it then compares to "GET " to check it's a HTTP GET.

# Troubleshooting
## SSL[^1][^2][^3]
Starting with Wireshark 2.0, the RSA key file is automatically matched against the public key as found in the Certificate handshake message. Before Wireshark 2.0, it relied on the user to enter a valid Address and Port value. Note that only RSA key exchanges can be decrypted using this RSA private key, Diffie-Hellman key exchanges cannot be decrypted using a RSA key file! (See "SSLKEYLOGFILE" if you have such a capture.)

连接建立的时候, 会告诉你是否采用Diffie Hellman key交换, 这样的方式你只能采用SSLKEYLOGFILE方式

## 抓不到应该出现的包
路由问题,路由问题,路由问题

重要的事情说三遍, 仔细检查路由表, 看看是否路由正确, 网络配置错误很容易导致.

## 关于tcp segment len超过1500
很可能是网卡支持tso, 同时打开了该功能, 应用层的拆包工作交给了网卡, 一般这种情况, 你能在对端看到正确的tcp segment len大小

## TCP Full Window
The Receiver advertises a TCP Window of 5000 byte.
The Sender sends 5 packets with a TCP Len of 1000 each
There is no ACK between those 5 packets
Wireshark will mark the 5th packet with [TCP Window Full] as it has seen those advertized 5000 bytes, without an ACK

## SACK
下面是一个wireshark包截图
![wireshark sack](/img/wireshark-sack.png)

截图信息显示，接收端希望接受的下一个包的sequence号是16953，但是它已经收到了18413~22793的包了，没有接收到16953~18412，发送端可以根据这个ack，重传16953~18412。


参考连接[TCP Selective Acknowledgments (SACK)](http://packetlife.net/blog/2010/jun/17/tcp-selective-acknowledgments-sack/)

## 如何查找一个session
标识一个tcp链接通过src_ip:port  -> dst_ip:port, 过滤条件写上就行, 当然tcp port存在重用的情况, 所以你还需要加上session id啥的, 在cookie里面有

# Usage
## 导出过滤的包
首先过滤包, 然后file -> export specified packets -> all packets
也可以通过tshark来过滤, 一般可以


# LUA
## httpext.lua

# 案例

## session失效
系统做了单点登录，用的是CAS，用户反馈偶尔系统登出之后，重新用其他帐号登入，查询
到的用户资料依然是上一个用户的，只有关闭浏览器之后，才能正常。

从直观上判断，应该是session问题，询问了一下应用开发人员，确认用户id存放在session
中。只有为啥session没有销毁成功，简单做法就是抓包分析。

先是用wireshark在客户端做了分析，发现在出现问题的时候，系统去请求出现问题的页面
不会做跳转（初次访问该页面需要去cas登入），也就是说对于业务系统来说，jsessionid
依然有效。

了解了一下cas的原理：用户去portal平台做登入，从cas获取到TGT。第一次访问业务模块
的时候，由于没有session，会创建一个session，然后重定向到sso做登入。而注销的时候
cas负责销毁创建的session。从故障看，问题就是出现在销毁session上。

通过tcpdump来抓包：
sso服务器端：
``` shell
[root@localhost ~]# tcpdump -p -nn -s 0 tcp port 8088 and host 172.16.102.205  -w sso.dump
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
```


应用端：
``` shell
[root@trust-test-app ~]# tcpdump -p -nn -s 0 host 172.17.102.106 and not tcp port 8001 -w sso-logout.dump
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
```

用wireshark分析包，可以看到CAS发出去的销毁session请求被reset，查看了一下环境，问
题出现在apache上，询问了一下，apache是自己编译的，而且配置文件也是乱七八糟，没想
明白为啥被reset而不是connection reject之类的，修改apache配置问题解决。

当然也不能算彻底解决，CAS关于注销的实现有bug，是一个不可靠实现，需要做一定处理。


[^1]: [Wireshark SSL](https://wiki.wireshark.org/SSL)

[^2]: [TLS完全指南](https://github.com/k8sp/tls)

[^3]: [Docker&Kubernetes tls guide](https://github.com/kelseyhightower/docker-kubernetes-tls-guide)
