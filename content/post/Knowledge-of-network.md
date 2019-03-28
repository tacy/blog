---
title: "Knowledge of network"
date: 2012-08-16
lastmod: 2012-08-16
draft: false
tags: ["tech", "network", "tcpip"]
categories: ["tech"]
description: "网络相关知识，基础介绍，这些记录主要是自己在学习网络虚拟化时候的一些细节，主要还是为了更好的了解网络虚拟化和其中的难点以及网络虚拟化目前的现状。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# tcp/ip

## 一些基本概念
ip在网络层，只负责发送和路由，不保证有序、正确、送达；tcp负责保证包有序、正确、送达，流量控制这种事自然也是tcp负责；端口说的是tcp层的事情，tcp是有状态的，而ip是无状态的，三次握手建立连接和4次握手关闭连接说的都是tcp的事情。

数据链路层（2层）不涉及到ip,如果要走路由，不是这一层能完成的，目前的很多虚拟网络都是基于2层，比如openstack quantum，vmware vds。
## 连接建立和关闭
client发起syn，server应答syn/ack，client应答ack，握手成功。
关闭连接可以由任意一端发起，假设client发起：client发送fin，server端应答fin/ack，客户端接收之后不再继续发送数据，server处理完剩余的数据之后，发送fin，客户端应答fin/ack，连接关闭。
## 连接状态

``` shell

     TCP A                                                TCP B
  1.  ESTABLISHED                                          ESTABLISHED
  2.  (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT
  3.  FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT
  4.                                                       (Close)
      TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK
  5.  TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --< CLOSED
  6.  (2 MSL)
      CLOSED
```

tcp A 发送FIN到tcp B，请求终止连接，tcp A来到FIN-WAIT-1状态；tcp B接收到FIN之后处于CLOSE-WAIT状态，tcp B发送ACK到TCP A；TCP A接收到ACK之后来到FIN-WAIT-2状态，这个时候，tcp A到tcp B的发送数据通道其实已经结束，但是由于连接是双向的，所以tcp A依然要等待tcp B发送完数据；tcp B处理完剩余的数据之后，发送FIN到tcp A，tcp B处于LAST-ACK状态，这个时候tcp A处于TIME-WAIT状态，tcp A应答ACK，tcp B CLOSED，tcp A等待2 MSL（一般是2毫秒）之后CLOSED。

## RTO
计算方法有很多种, 首先是采样RTT, 然后计算SRTT, 计算SRTT的公式:
`SRTT = (SRTT*0.9) + RTT*0.1`, TCPIP只需要记住一个旧的SRTT即可, 新采样的RTT比例占0.1~0.2

然后计算RTO: `RTO = min(ubound, max(lbound, SRTT*(1.3~2))`, ubound代表上线, 建议设置, 例如1分钟, 同样, lbound建议设置, 例如1秒.

## min rto
The minimum RTO can be changed. TCP_RTO_MIN, which is a kernel configu-
ration constant, can be changed prior to recompiling and installing the kernel.
Some Linux versions also allow it to be changed using the ip route command.
When TCP is used in data-center networks where RTTs may be a few microsec-
onds, 200ms minimum RTO can lead to severe performance degradations due to
slow TCP recovery after packet loss in local switches. This is the so-called TCP
\u201cincast\u201d problem. Various solutions exist to this problem, including modification of
the TCP timer granularity and minimum RTO to be on the order of microseconds
[V09]. Such small minimum RTO values are not recommended for use on the
global Internet.

## tcp_fin_timeout
主动断开连接方，会等待对方回FIN包，并且处于FIN-WAIT-2状态，tcp_fin_timeout用于控制FIN-WAIT-2状态的时间，如果超过设定时间没有接收到FIN包，回直接close连接，连接进入time_wait状态


## Fast Retransmit

一般如果接受端收到乱序的包, 会立即发送dupack, 但是如果网络异常也会导致dupack, 所以发送端会等到duplicate ACK threshold or dupthresh达到, 然后就会重传包, 如果不支持sack, 一次只会重传一个包, 然后继续发送新包

如果支持sack, 可以一次触发多个包重传.

## Receiver Windows
如果sequence number小于left edge, 丢弃(重传包), 如果大于right edge, 丢弃(不应该出现的包, 无法保存).

reveiver只会ack连续的left edge的包, 如果不连续的包接收到, 会通过sack通知sender(out-of-order)

## Windows size
TCP window scale option is needed for efficient transfer of data when the bandwidth-delay product (BDP) is greater than 64K. For instance, if a T1 transmission line of 1.5 Mbit/second was used over a satellite link with a 513 millisecond round trip time (RTT), the bandwidth-delay product is (1,500,000 * 0.513) = 769,500 bits or about 96,187 bytes. Using a maximum buffer size of 64 KiB only allows the buffer to be filled to (65,535 / 96,187) = 68% of the theoretical maximum speed of 1.5 Mbits/second, or 1.02 Mbit/s.

For example if the receive window is 2^15=32,768 bytes(receive window is the 2 bytes in the TCP header). If the window scale factor(options in TCP) is 3, calculation is as follows: (2^15) * (2^3) = 262,144 bytes. In essence, this is 2^18 = 262,144. (Shifting 3 bits to the left) Note that in the options field, window scale factor is sent in the SYN packet. Also, the value of the Window scale factor can go until 255 (2^8 = 1 byte) but the maximum allowed is 14. The reason why maximum value is 14 is because (2^16)*(2^14) = 1,073,741,824. If it increases beyond this then it will surpass the Sequence no#. Sequence no# is 4 bytes (2^32) and the size of the window can't go beyond the maximum value of the sequence no#.[1]

By using the window scale option, the receive window size may be increased up to a maximum value of 1,073,725,440 bytes. This is done by specifying a one byte shift count in the header options field. The true receive window size is left shifted by the value in shift count. A maximum value of 14 may be used for the shift count value. This would allow a single TCP connection to transfer data over the example satellite link at 1.5 Mbit/second utilizing all of the available bandwidth.

Essentially, not more than one full transmission window can be transferred within one round-trip time period. The window scale option enables a single TCP connection to fully utilize an LFN with a BDP of up to 1 GB, e.g. a 10 Gbit/s link with round-trip time of 800 ms.

Linux kernels (from 2.6.8, August 2004) have enabled TCP Window Scaling by default. The configuration parameters are found in the /proc filesystem, see pseudo-file /proc/sys/net/ipv4/tcp_window_scaling and its companions /proc/sys/net/ipv4/tcp_rmem and /proc/sys/net/ipv4/tcp_wmem (more information: man tcp, section sysctl).[7]

Scaling can be turned off by issuing the command sysctl -w "net.ipv4.tcp_window_scaling=0" as root. To maintain the changes after a restart, include the line "net.ipv4.tcp_window_scaling=0" in /etc/sysctl.conf (or /etc/sysctl.d/99-sysctl.conf as of systemd 207).
### RWND
TODO
### CWND
拥塞窗口, 一般tcp/ip能明确知道两端的buffer windows大小(ack/syn都携带窗口信息), 但是不能知道链路的拥塞情况, 所以在开始的时候会使用slow start机制, 也就是先试探着发送几个MSS, 如果情况良好, 继续增加CWND, 发送几个MSS由initcwnd控制, 该参数可以调整:

``` shell
shell> ip route | while read r; do
           ip route change $r initcwnd 10;
       done
```

## MSS
只计算data, 不计算header, SYN会携带MSS信息, 如果没有MSS携带, 默认是536(IP数据包包大小+最小的ip头+最小的tcp头结果不能小于576, 这是规范定义的)

## SACK (Selective Acknowledgment)
In Chapter 12 we introduced the concept of a sliding window, and we described how TCP handles its sequence numbers and acknowledgments. Because it uses cumulative ACKs, TCP is never able to acknowledge data it has received correctly but that is not contiguous, in terms of sequence numbers, with data it has received previously. In such cases, the TCP receiver is said to have holes in its received data queue. A receiving TCP prevents applications from consuming data beyond a hole because of the byte stream abstraction it provides.

If a TCP sender were able to learn of the existence of holes (and out-of-sequence data blocks beyond holes in the sequence space) at the receiver, it could better select which particular TCP segments to retransmit when segments are lost or otherwise missing at the receiver. The TCP selective acknowledgment (SACK) options [RFC2018][RFC2883] provide this capability. The scheme works effectively, however, only if the TCP sender logic is able to make effective use of the SACK information it receives from a SACK-capable receiver.

A TCP learns that its peer is capable of advertising SACK information by receiving the SACK-Permitted option in a SYN (or SYN + ACK) segment. Once this has taken place, the TCP receiving out-of-sequence data may provide a SACK option that describes the out-of-sequence data to help its peer perform retransmissions more efficiently. SACK information contained in a SACK option consists of a range of sequence numbers representing data blocks the receiver has successfully received. Each range is called a SACK block and is represented by a pair of 32-bit sequence numbers. Thus, a SACK option containing n SACK blocks is (8n + 2) bytes long. Two bytes are used to hold the kind and length of the SACK option.

## ACK
ack应答, 会告诉对端, 之前的包我都收到了, 这样能应对ack包丢包的情况
When TCP receives data from the other end of the connection, it sends an acknowledgment. This acknowledgment may not be sent immediately but is normally delayed a fraction of a second. The ACKs used by TCP are cumulative in thesense that an ACK indicating byte number N implies that all bytes up to number N (but not including it) have already been received successfully. This provides some robustness against ACK loss\u2014if an ACK is lost, it is very likely that a subsequent ACK is sufficient to ACK the previous segments.

syn和fin包都需要占用一个sequence number, 而在tcp中, 需要消耗sequence number的包都需要重传机制, 而ack不消耗, 无需重传
When a new connection is being established, the SYN bit field is turned on in the first segment sent from client to server. Such segments are called SYN segments,or simply SYNs. The Sequence Number field then contains the first sequence number to be used on that direction of the connection for subsequent sequence numbers and in returning ACK numbers (recall that connections are all bidirectional). Note that this number is not 0 or 1 but instead is another number, often randomly chooen, called the initial sequence number (ISN). The reason for the ISN not being 0 or 1 is a security measure and will be discussed in Chapter 13. The sequence number of the first byte of data sent on this direction of the connection is the ISN plus 1 because the SYN bit field consumes one sequence number. As we shall see later,consuming a sequence number also implies reliable delivery using retransmission.Thus, SYNs and application bytes (and FINs, which we will see later) are reliably delivered. ACKs, which do not consume sequence numbers, are not.

# Hub & Bridge & Switch & Router
Hub工作在第一层（物理层），Bridge工作在第二层（数据链路层）；Switch分很多种，一般我们说的Switch也是工作在第二层，但是也有很多Switch工作在三层、四层、四-七层；Router工作在三层。

Hub没有学习能力，只是一个中继，他收到一个packet之后，直接广播到所有的其他端口，由于带宽共享和处于单一包冲突域（collision domain），基本无法实现大量主机高速连接。

Switch具备MAC学习能力，每个端口为一个冲突域，每个通讯建立独立的链路，全速带宽。能有效建立高速网络。三层交换具备路由功能，四层具备负载均衡能力，四-七层CDN。

Bridge工作在二层，因此可以连接异构网络和多协议，其他感觉和两层Switch差不多，当然没有Switch功能那么多，更多是用来在网络间连接。当然linux bridge相比硬件bridge更强大，他能过滤和改变传输，具体可以参考ebtables

Router工作在三层（网络层），实现路由功能。

# VLAN
vlan工作在二层，早期vlan是解决大的单一网络的冲突域问题，现在主要解决mac层的广播域问题。vlan id 设计为12位，支持4096个
vlan。一般一个vlan可以对应多个ip子网。trunk一般是指switch-to-switch,switch-to-router，而不是指连接到主机的端口。

# tunnel

## vxlan
Vxlan是cisco、Vmware等发起的一个标准，主要解决Vlan可扩展性和基于二层的局限性，它采用MAC-in-UDP技术，封装两层的包到三层，工作在三层，主要应用在当前的云计算中心虚拟网络，可以划分16M的网络（24位id），同时实现虚拟机跨两层网络迁移，目前采用的都是one-on-one方式，不支持multipath。简单概括vxlan：实现跨三层的两层网络，利用现有的路由

## otv (Overlay Transport vritualization)
解决跨数据中心实现两层网络扩展（异地，跨互联网），vxlan的包封装格式和otv类似，解决问题不一样，实现Vlan跨数据中心。cisco最晚版本不需要依赖IP Multicast

# TAP/TUN
1. What is the TUN ?
The TUN is Virtual Point-to-Point network device. TUN driver was designed as low level kernel support for IP tunneling. It provides to userland application two interfaces:
  - /dev/tunX	- character device;
  - tunX	- virtual Point-to-Point interface.

Userland application can write IP frame to /dev/tunX and kernel will receive this frame from tunX interface. In the same time every frame that kernel writes to tunX interface can be read by userland application from /dev/tunX device.

2. What is the TAP ?
The TAP is a Virtual Ethernet network device. TAP driver was designed as low level kernel support for Ethernet tunneling. It provides to userland application two interfaces:
  - /dev/tapX	- character device;
  - tapX	- virtual Ethernet interface.

Userland application can write Ethernet frame to /dev/tapX and kernel will receive this frame from tapX interface. In the same time every frame that kernel writes to tapX interface can be read by userland application from /dev/tapX device.

简单说tun工作在IP层，走路由，tap工作在数据链路层，tap其实就是一个虚拟网卡，bridge可以利用它实现异地互联（局域网）

# 网络虚拟化

面临的问题：
1. 多租户网络管理问题，目前通常采用vlan实现流量隔离，受限于VLAN的4096问题，租户受限，另外租户内流量隔离也无法满足。
2. Switch mac 表空间问题，虚拟环境下，虚拟机导致mac地址激增，容易导致mac表空间溢出，switch会停止mac地址学习，导致网络风暴。
3. 两层网络的STP（spanning tree protocol）问题，导致浪费和多路径支持问题。
4. 独立于物理网络的虚拟机迁移。

# Overlay

网络虚拟化目前是一个很热门的话题，主要的话题集中在SDN和Overlay，这里是对overlay的一个简单介绍

Overlay的提出主要是要解决数据中心网络虚拟化问题，在这之前有又很多这方面的努力，比如OTV,LISP等等，实现跨数据中心VLAN延伸和ip位置分离，这些技术也可以很好结合。

## 四个特性
1. 多租户流量隔离
2. 租户有自己独立的ip网络
3. 允许VM位置自由和任意迁移而不受底层物理网络结构限制
4. 好的扩展性（支持更多的租户）

## 主流的实现方式
### vxlan
采用mac-over-udp方式，实现跨三层的两层虚拟网络，由cisco和vmware提出，实现机制类似Bridge（flood和learn）

优点：
1. 不用修改底层网络配置
2. 支持VM数据内中心迁移
3. 支持16M网段
4. 支持PortChannel和ACL

缺点：
1. 需要ip multicast支持，这也决定了他无法跨wan
2. 现有的网络服务无法使用，比如LB，Firewall等，目前这些都需要通过特定VM实现（比如Vmware Vshield Edge），影响性能
3. 由于在三层实现，无法使用网卡的Offload特性（tso、lro）

Vxlan跨Router的时候，需要首跳Router打开ARP Proxy，目前已经有支持Vxlan的switch计划上市，我想支持tunnel的网卡应该也快了

### nvgre
采用mac-over-gre方式，实现跨三层的两层虚拟网络，由microsoft等厂商提出

优点：
1. 不用修改底层网络配置
2. 支持VM数据中心内迁移
3. 支持16M网段

缺点：
1. 由于很多switch和router不支持GRE解析，无法支持一些load distribution（Port Channel）和安全策略（ACLS）
2. 其他同Vxlan

### stt
采用mac-over-ip方式，增加了一个tcp header，具体没看太明白，由nicira提出

优点
1. 支持网卡offload
2. 其他同Vxlan

缺点
1. 需要ip multicast支持，这也决定了他无法跨wan
2. 现有的网络服务无法使用，比如LB，Firewall等，目前这些都需要通过特定VM实现（比如Vmware Vshield Edge），影响性能

# Trill
The goal of trill is combine the best aspects of L2 with the best aspects of L3, The means easy configuration, "Plug and play", multipathing(ECMP), fast convergence, and easy scalability.

# LISP (locate identify split protocol)

# OVT

# layer 2 network & layer 3 network
1. layer 2 stp问题，浪费物理链路，且不支持多路径，layer 3采用路由，支持多路径
2. layer 2 boardcast frame问题，对于vdc，强制hypervisor nic处于promiscuous模式，接受所有的boardcast frame；另外太多的vm，导致switch mac表溢出，形成广播风暴
3. layer 2 vlan限制，只能支持4094个vlan，子网可以任意
4. layer 2 故障域问题，影响layer 2域（segment）甚至整个网络，layer 3只影响具体subnet

# 参考资料
1. [[http://www.frozentux.net/iptables-tutorial/iptables-tutorial.html][iptables tutorial]]
2. [[http://en.wikipedia.org/wiki/Ethernet_hub][Ethernet hub]]
3. [[http://www.linuxfoundation.org/collaborate/workgroups/networking/bridge][linux bridge]]
4. [[https://learningnetwork.cisco.com/servlet/JiveServlet/previewBody/2810-102-1-7611/primch5.pdf][Network interfaces, hubs,switches, bridges, routers, and firewalls]]
5. [[http://en.wikipedia.org/wiki/Multilayer_switch][Multilayer switch]]
6. [[http://www.cisco.com/en/US/prod/collateral/switches/ps9441/ps9902/white_paper_c11-685115.html][Scalable Cloud Networking with Cisco Nexus 1000V Series Switches and VXLAN]]
7. [[http://www.kernel.org/pub/linux/kernel/people/marcelo/linux-2.4/Documentation/networking/tuntap.txt][tun/tap]]
8. [[http://openvpn.net/index.php/open-source/faq.html][openvpn faq]]
9. [[http://tools.ietf.org/html/draft-mahalingam-dutt-dcops-vxlan-00][vxlan draft]]
10. [[http://it20.info/2012/05/typical-vxlan-use-case/][Typical vxlan use case]]
