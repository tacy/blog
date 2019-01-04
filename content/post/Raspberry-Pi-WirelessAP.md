---
title: "Raspberry Pi WirelessAP"
date: 2014-10-29
lastmod: 2014-10-29
draft: false
tags: ["tech", "linux", "raspberrypi"]
categories: ["tech"]
description: "把Raspberry Pi改造成无线AP, 改造起来也很简单, 记录一下过程"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Raspberry Pi WirelessAP
有时候, 希望能做一些其他设备的抓包分析, 不想去折腾路由器, 就把手上的Raspberry Pi改造了一下, 当然你也能让他当成翻墙工具, 可以自己扩展.

网上已经有一大堆文档, 大部分都已经过期了, 所以建议大家在搜索的时候, 尽量选择近期的文档, 这样会比较靠谱一点.

拿我的设备举例子: 我的无线网卡是RTL8192CU, 网上的大部分帖子都过期了, 上来就让你编译网卡驱动. 其实这个网卡在kernel 3.10之前有bug, 无法正确支持, 但是在后续版本中已经fix了, 我现在用的kernel是3.11, 自动支持.

主要的问题是hostapd对RT8192CU的支持有点问题, 需要用hack过的版本.

## 环境准备
无线网卡: RT8192CU (正常的话, lsusb能识别, 也能通过dmesg查看)
kernel: 3.11
安装相关软件
: sudo apt-get install hostapd isc-dhcp-server

## 配置无线网卡和网络路由
配置无线网卡为静态地址, 当然不要和eth0地址段冲突了, 如果你家里是192.168.1.0/24, 那么这里你需要设置为该网段以外的其他网段.
``` shell
pi@raspbmc:/etc/dhcp$ cat /etc/network/interfaces
auto wlan0
allow-hotplug wlan0
iface wlan0 inet static
    address 192.168.42.1
    netmask 255.255.255.0
up iptables-restore < /etc/iptables.ipv4.nat
```

接下来你需要启用ip转发功能, 简单做法是编辑/etc/sysctl.conf, 找到下面这样, 取消注释.
: #net.ipv4.ip_forward=1
然后通过下面命令启用:
: sysctl -p

设置NAT规则, 让包能够正常路由到其他网段:
``` shell
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```

保存iptables配置:
: sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"

## 配置DHCP服务
你需要给连接AP的客户端自动分配ip地址信息, 所有你需要有dhcp服务, 当然你也能手动制定, 但是对于客户端太麻烦了.

先编写dhcp配置文件, 主要是注释下面两行:
``` shell
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;
```

取消注释下面一行:
: #authoritative;

添加下面内容到文件结尾, 注意这里的ip地址段和无线网卡匹配:
``` shell
subnet 192.168.42.0 netmask 255.255.255.0 {
    range 192.168.42.10 192.168.42.50;
    option broadcast-address 192.168.42.255;
    option routers 192.168.42.1;
    default-lease-time 600;
    max-lease-time 7200;
    option domain-name "local";
    option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

完整内容如下:
``` shell
pi@raspbmc:/etc/dhcp$ cat dhcpd.conf
#
# Sample configuration file for ISC dhcpd for Debian
#
#

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# option definitions common to all supported networks...
#option domain-name "example.org";
#option domain-name-servers ns1.example.org, ns2.example.org;

default-lease-time 600;
max-lease-time 7200;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
log-facility local7;

# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.

#subnet 10.152.187.0 netmask 255.255.255.0 {
#}

# This is a very basic subnet declaration.

#subnet 10.254.239.0 netmask 255.255.255.224 {
#  range 10.254.239.10 10.254.239.20;
#  option routers rtr-239-0-1.example.org, rtr-239-0-2.example.org;
#}

# This declaration allows BOOTP clients to get dynamic addresses,
# which we don't really recommend.

#subnet 10.254.239.32 netmask 255.255.255.224 {
#  range dynamic-bootp 10.254.239.40 10.254.239.60;
#  option broadcast-address 10.254.239.31;
#  option routers rtr-239-32-1.example.org;
#}

# A slightly different configuration for an internal subnet.
#subnet 10.5.5.0 netmask 255.255.255.224 {
#  range 10.5.5.26 10.5.5.30;
#  option domain-name-servers ns1.internal.example.org;
#  option domain-name "internal.example.org";
#  option routers 10.5.5.1;
#  option broadcast-address 10.5.5.31;
#  default-lease-time 600;
#  max-lease-time 7200;
#}

# Hosts which require special configuration options can be listed in
# host statements.   If no address is specified, the address will be
# allocated dynamically (if possible), but the host-specific information
# will still come from the host declaration.

#host passacaglia {
#  hardware ethernet 0:0:c0:5d:bd:95;
#  filename "vmunix.passacaglia";
#  server-name "toccata.fugue.com";
#}

# Fixed IP addresses can also be specified for hosts.   These addresses
# should not also be listed as being available for dynamic assignment.
# Hosts for which fixed IP addresses have been specified can boot using
# BOOTP or DHCP.   Hosts for which no fixed address is specified can only
# be booted with DHCP, unless there is an address range on the subnet
# to which a BOOTP client is connected which has the dynamic-bootp flag
# set.
#host fantasia {
#  hardware ethernet 08:00:07:26:c0:a5;
#  fixed-address fantasia.fugue.com;
#}

# You can declare a class of clients and then do address allocation
# based on that.   The example below shows a case where all clients
# in a certain class get addresses on the 10.17.224/24 subnet, and all
# other clients get addresses on the 10.0.29/24 subnet.

#class "foo" {
#  match if substring (option vendor-class-identifier, 0, 4) = "SUNW";
#}

#shared-network 224-29 {
#  subnet 10.17.224.0 netmask 255.255.255.0 {
#    option routers rtr-224.example.org;
#  }
#  subnet 10.0.29.0 netmask 255.255.255.0 {
#    option routers rtr-29.example.org;
#  }
#  pool {
#    allow members of "foo";
#    range 10.17.224.10 10.17.224.250;
#  }
#  pool {
#    deny members of "foo";
#    range 10.0.29.10 10.0.29.230;
#  }
#}

subnet 192.168.42.0 netmask 255.255.255.0 {
    range 192.168.42.10 192.168.42.50;
    option broadcast-address 192.168.42.255;
    option routers 192.168.42.1;
    default-lease-time 600;
    max-lease-time 7200;
    option domain-name "local";
    option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

接下来需要告诉dhcp, 它需要绑定在哪个网卡上, 简单修改isc-dhcp-server文件, 设置INTERFACES="wlan0":
``` shell
pi@raspbmc:/etc/dhcp$ cat /etc/default/isc-dhcp-server
# Defaults for isc-dhcp-server initscript
# sourced by /etc/init.d/isc-dhcp-server
# installed at /etc/default/isc-dhcp-server by the maintainer scripts

#
# This is a POSIX shell fragment
#

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPD_CONF=/etc/dhcp/dhcpd.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPD_PID=/var/run/dhcpd.pid

# Additional options to start dhcpd with.
#	Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#	Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACES="wlan0"
```

## 配置hostapd

编写hostapd.conf文件, 内容如下:
``` shell
pi@raspbmc:/etc/hostapd$ cat hostapd.conf
interface=wlan0
driver=rtl871xdrv
ssid=Pi_AP
hw_mode=g
channel=6
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=Raspberry
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

修改hostapd.conf文件, 指定配置文件位置DAEMON_CONF="/etc/hostapd/hostapd.conf":
``` shell
pi@raspbmc:/etc/hostapd$ cat /etc/default/hostapd
# Defaults for hostapd initscript
#
# See /usr/share/doc/hostapd/README.Debian for information about alternative
# methods of managing hostapd.
#
# Uncomment and set DAEMON_CONF to the absolute path of a hostapd configuration
# file and hostapd will be started during system boot. An example configuration
# file can be found at /usr/share/doc/hostapd/examples/hostapd.conf.gz
#
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

用下面hack过的hostapd替换原始的:
: wget http://www.adafruit.com/downloads/adafruit_hostapd.zip

用解压之后的文件替换/usr/sbin/hostapd, 记得添加执行权限, 替换之前也可以备份一下原始版本

完工, 现在检查一下效果, 确认:
1. 无线网卡配置正常(ifconfig, wlan0正常)
2. 确认ip_forward启用(/proc/sys/net/ipv4/ip_forward为1)
3. 确认iptables nat正常(sudo iptables -t nat -S)
3. dhcp服务启动(sudo service dhcp restart)
4. hostapd服务启动(sudo service hostapd restart)

正常的话, 你在客户端应该能看到一个叫Pi_AP的信号了!
