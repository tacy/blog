---
title: "l2tp ipsec in linode"
date: 2014-12-24
lastmod: 2014-12-24
draft: false
tags: ["tech", "linux", "vpn"]
categories: ["tech"]
description: "l2tp ipsec 在linode上碰到的问题"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

我参考的配置帖子: https://raymii.org/s/tutorials/IPSEC_L2TP_vpn_with_Ubuntu_14.04.html

先注意下面问题:
# Two or more interfaces found, checking IP forwarding Failed
我的环境是: My Ubuntu 14.04 LTS Profile (Latest 64 bit (3.16.7-x86_64-linode49))

如果已经配置了ip_forward, 没啥好说的, 直接升级openswan吧, ppa在这里:https://launchpad.net/~openswan/+archive/ubuntu/ppa

如果你用的是和我一样的配置, 那么先添加上面的ppa(貌似12.04没有上面的问题)

然后按照下面的步骤操作:

# 安装软件包
``` shell
apt-get update
apt-get install openswan xl2tpd ppp lsof
```

# 配置网络环境
配置IP转发禁用ICP重定向
``` shell
echo "net.ipv4.ip_forward = 1" |  tee -a /etc/sysctl.conf
echo "net.ipv4.conf.all.accept_redirects = 0" |  tee -a /etc/sysctl.conf
echo "net.ipv4.conf.all.send_redirects = 0" |  tee -a /etc/sysctl.conf
echo "net.ipv4.conf.default.rp_filter = 0" |  tee -a /etc/sysctl.conf
echo "net.ipv4.conf.default.accept_source_route = 0" |  tee -a /etc/sysctl.conf
echo "net.ipv4.conf.default.send_redirects = 0" |  tee -a /etc/sysctl.conf
echo "net.ipv4.icmp_ignore_bogus_error_responses = 1" |  tee -a /etc/sysctl.conf
```

其他网卡也来一遍:
``` shell
for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done
```

最好让上面配置生效:
: sysctl -p

配置iptables nat规则:
: iptables -t nat -A POSTROUTING -j SNAT --to-source YOUR_PUBLIC_IP -o eth0

上面两条会在重启之后丢失, 为了让他们重启能生效, 放到rc.local里面(注意下面内容添加到 exit 0 这行前面):
``` shell
for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done

iptables -t nat -A POSTROUTING -j SNAT --to-source YOUR_PUBLIC_IP -o eth0

```

# 配置Openswan(IPSEC)

修改/etc/ipsec.conf文件, 用下面内容替换:
``` shell
version 2 # conforms to second version of ipsec.conf specification

config setup
    dumpdir=/var/run/pluto/
    #in what directory should things started by setup (notably the Pluto daemon) be allowed to dump core?

    nat_traversal=yes
    #whether to accept/offer to support NAT (NAPT, also known as "IP Masqurade") workaround for IPsec

    virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v6:fd00::/8,%v6:fe80::/10
    #contains the networks that are allowed as subnet= for the remote client. In other words, the address ranges that may live behind a NAT router through which a client connects.

    protostack=netkey
    #decide which protocol stack is going to be used.

    force_keepalive=yes
    keep_alive=60
    # Send a keep-alive packet every 60 seconds.

conn L2TP-PSK-noNAT
    authby=secret
    #shared secret. Use rsasig for certificates.

    pfs=no
    #Disable pfs

    auto=add
    #the ipsec tunnel should be started and routes created when the ipsec daemon itself starts.

    keyingtries=3
    #Only negotiate a conn. 3 times.

    ikelifetime=8h
    keylife=1h

    ike=aes256-sha1,aes128-sha1,3des-sha1
    phase2alg=aes256-sha1,aes128-sha1,3des-sha1
    # https://lists.openswan.org/pipermail/users/2014-April/022947.html
    # specifies the phase 1 encryption scheme, the hashing algorithm, and the diffie-hellman group. The modp1024 is for Diffie-Hellman 2. Why 'modp' instead of dh? DH2 is a 1028 bit encryption algorithm that modulo's a prime number, e.g. modp1028. See RFC 5114 for details or the wiki page on diffie hellmann, if interested.

    type=transport
    #because we use l2tp as tunnel protocol

    left=YOUR_PUBLIC_IP
    #fill in server IP above

    leftprotoport=17/1701
    right=%any
    rightprotoport=17/%any

    dpddelay=10
    # Dead Peer Dectection (RFC 3706) keepalives delay
    dpdtimeout=20
    #  length of time (in seconds) we will idle without hearing either an R_U_THERE poll from our peer, or an R_U_THERE_ACK reply.
    dpdaction=clear
    # When a DPD enabled peer is declared dead, what action should be taken. clear means the eroute and SA with both be cleared.
```

注意把YOUR_PUBLIC_IP替换成你的服务器ip地址.

设置你的shared secret:

编辑/etc/ipsec.secrets文件, 添加下面行:

: YOUR_PUBLIC_IP %any: PSK "yousharedsecret"

重启ipsec服务:

: service ipsec restart

验证配置:
: ipsec verify

你应该看到下面输出:
``` shell
Checking if IPsec got installed and started correctly:

Version check and ipsec on-path                   	[OK]
Openswan U2.6.42/K3.16.7-x86_64-linode49 (netkey)
See `ipsec --copyright' for copyright information.
Checking for IPsec support in kernel              	[OK]
 NETKEY: Testing XFRM related proc values
         ICMP default/send_redirects              	[OK]
         ICMP default/accept_redirects            	[OK]
         XFRM larval drop                         	[OK]
Hardware random device check                      	[N/A]
Two or more interfaces found, checking IP forwarding	[OK]
Checking rp_filter                                	[ENABLED]
 /proc/sys/net/ipv4/conf/all/rp_filter            	[ENABLED]
Checking that pluto is running                    	[OK]
 Pluto listening for IKE on udp 500               	[OK]
 Pluto listening for IKE on tcp 500               	[NOT IMPLEMENTED]
 Pluto listening for IKE/NAT-T on udp 4500        	[OK]
 Pluto listening for IKE/NAT-T on tcp 4500        	[NOT IMPLEMENTED]
 Pluto listening for IKE on tcp 10000 (cisco)     	[NOT IMPLEMENTED]
Checking NAT and MASQUERADEing                    	[TEST INCOMPLETE]
Checking 'ip' command                             	[OK]
Checking 'iptables' command                       	[OK]
```

# 配置xl2tpd
编辑xl2tpd配置文件/etc/xl2tpd/xl2tpd.conf, 用下面内容替换:
``` shell
[global]
ipsec saref = yes
saref refinfo = 30

;debug avp = yes
;debug network = yes
;debug state = yes
;debug tunnel = yes

[lns default]
ip range = 172.16.1.30-172.16.1.100
local ip = 172.16.1.1
refuse pap = yes
require authentication = yes
;ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
```

# 配置PPP
编辑ppp配置文件/etc/ppp/options.xl2tpd, 用下面内容替换, 如果该配置文件不存在, 手动创建它
``` shell
require-mschap-v2
ms-dns 8.8.8.8
ms-dns 8.8.4.4
auth
mtu 1200
mru 1000
crtscts
hide-password
modem
name l2tpd
proxyarp
lcp-echo-interval 30
lcp-echo-failure 4
```

注意其中的mtu和mru别随便改动, 我调到1400网络性能急剧下降, 分析了一下, 很多包重发现象, 当然如果你熟悉sniffer, 可以自己调整一下, 估计1300什么的应该是可以的.

添加用户, 在/etc/ppp/chap-secrets文件中填加用户:
``` shell
# Secrets for authentication using CHAP
# client       server  secret                  IP addresses
alice          l2tpd   0F92E5FC2414101EA            *
bob            l2tpd   DF98F09F74C06A2F             *
```

上面就是你未来客户端拨号用的账户信息

现在重启ipsec和xl2tpd服务:
``` shell
service ipsec restart
/etc/init.d/xl2tpd restart
```

配置完成了

# 客户端配置

我用的是ubuntu 12.04 , 直接用network-manager-l2tp插件就行. 至于其他的客户端, 直接google吧
