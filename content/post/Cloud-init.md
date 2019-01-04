---
title: "在本地kvm环节下使用cloud image"
date: 2014-11-07
lastmod: 2014-11-07
draft: false
tags: ["tech", "virtualization", "kvm"]
categories: ["tech"]
description: "大量的cloud image, 其实在本地虚拟环境也能使用, 这里简单介绍一下在我本机kvm环境下使用cloud image的步骤"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# 在本地kvm环境下使用cloud image

大量的cloud image, 其实在本地虚拟环境也能使用, 这里简单介绍一下在我本机kvm环境下使用cloud image的步骤

这里用的是openvswitch做网络环境配置

## 配置openvswitch+kvm
``` shell

apt-get install qemu-kvm openvswitch-switch

vi /etc/network/interface
auto ovsbr0
iface ovsbr0 inet static
   address 172.16.11.1
   network 172.16.11.0
   netmask 255.255.255.0
   broadcast 172.16.11.255

ovs-vsctl add-br ovsbr0

/etc/openvswitch/ovs-ifup
#!/bin/sh
switch='ovsbr0'
/sbin/ifconfig $1 0.0.0.0 up
ovs-vsctl add-port ${switch} $1

/etc/openvswitch/ovs-ifdown
#!/bin/sh
switch='ovsbr0'
/sbin/ifconfig $1 0.0.0.0 down
ovs-vsctl del-port ${switch} $1

```

## 配置ubuntu cloud image

去[[https://cloud-images.ubuntu.com/releases/14.04.1/release/][Ubuntu网站]] 下载相对应的image, 我这里用的是1404做测试:
``` shell
tacy@momo1:~/$ wget https://cloud-images.ubuntu.com/releases/14.04.1/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img

tacy@momo1:~/$ tar zxvf ubuntu-14.04-server-cloudimg-amd64-disk1.img
```

这是一个已经安装好的ubunt 14.04镜像, 文件格式是qcow2, 磁盘空间大小是2.2G, 可以用qemu-img resize.

: tacy@momo1:~/$ qemu-img resize ubuntu-14.04-server-cloudimg-amd64-disk1.img +8g

然后我们做一个快照, 用它来启动虚拟机

: tacy@momo1:~/$ qemu-img create -f qcow2 -b ubuntu-14.04-server-cloudimg-amd64-disk1.img node01.img

准备好image, 现在我们来配置本地的cloud init, 让cloud image能启动起来, 也很简单, 建立下面两个文件:

``` shell
tacy@momo1:~/$ mkdir metadata && cd metadata
tacy@momo1:~/metadata$ cat meta-data
instance-id: iid-local01
local-hostname: cloudimg
network-interfaces: |
  auto eth0
  iface eth0 inet static
  address 172.16.11.100
  network 172.16.11.0
  netmask 255.255.255.0
  broadcast 172.16.11.255
  gateway 172.16.11.1

tacy@momo1:~/metadata$ cat user-data
#cloud-config
#password: passw0rd
#chpasswd: { expire: False }
#ssh_pwauth: True
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUtDcwkNskzigh0rjirK/BoualyOiuZoBLMpK03MHBUPh/wHgoiSGOUoSGY7RMcVclKECqCQjyv3WN6vJnDwjQ1mEiXFKntnWqqJYZsDETttDfxYwPmV2sA5UBfSFDUuBhmLYVsJg/T9NUf/K/aO8RI2Q7M09Xds6hfilO1rR59h/8/d3fbj8QG/DBnEFe6HxQj7OX5RGPbL/dT9OlLdDLhRf6rPHFHVy7PKTU1SfzIsL89v9MXkAcet+zb5UJcuifSMIQQhSv8MhhWscZibkXQi1btqxNgoxIVguW57fghR7wpVUn6oAHiCnz3KY34N8Nv1UodY6kk4idUmQ0oJiZ xx@xx

```

这里的ssh key是你的公钥, 当然也可以配置密码登陆.

生成镜像文件:
: genisoimage  -output seed.iso -volid cidata -joliet -rock user-data meta-data

## 启动虚拟机
现在可以启动虚拟机了:
``` shell

sudo kvm -m 2048m -smp 2 \
-drive file=node01.img,if=virtio,cache=none,aio=native \
-drive file=./metadata/seed.iso,if=virtio \
-net nic,vlan=0,model=virtio,macaddr=DE:AD:BE:EF:FC:80 \
-net tap,vlan=0,vhost=on,script=/etc/openvswitch/ovs-ifup,downscript=/etc/openvswitch/ovs-ifdown \
-nographic -curses
```

启动之后, 你的配置信息注入到了虚拟机里面, 你现在可以直接访问了:

: ssh ubuntu@172.16.11.100

想象一下, 这样你自己完全可以diy一个单机的IaaS平台嘛!

当然这里没有配置虚拟机的网络访问, 如果要实现网络访问, 需要配置虚拟机的resolv.conf和宿主机的路由转发.

## 宿主机路由
首先打开转发功能, 编辑/etc/sysctl.conf, 打开ip_forward=1那一行, 同时让配置生效:
: sysctl -p

配置iptables:
``` shell
iptables -I FORWARD -j ACCEPT
iptables -t nat -A POSTROUTING -s 172.16.11.0/24 ! -d 172.168.11.0/24 -j MASQUERADE
```

这里只是测试环境, 所以我打开了所有包的转发, 当然你也能细粒度控制包转发.

# CoreOS的cloud config
coreos这是个顺应容器时代的操作系统, 他和其他linux发行版完全不一样, 没有提供包管理机制, 预编译升级, 支持很多IaaS平台, 如果你没有IaaS平台, 当然你也能在本地用, 社区有支持版本[fn:1]. 我一般在自己机器上用kvm, 测试了一下, 非常方便, 简单注意几点:

## cloud config
该文件必须存放在特殊位置, 使用方式如下:
``` shell
mkdir -p myconfig/openstack/latest
cp user_data myconfig/openstack/latest/
```
user_data就是coreos的cloud config文件, 具体用法请参考CoreOS文档[fn:2]. 必须存放在openstack/latest目录下才行.

使用方式在kvm启动命令中添加如下启动项:
``` shell
-fsdev local,id=conf,security_model=none,readonly,path=~/myconfig \
-device virtio-9p-pci,fsdev=conf,mount_tag=config-2 \
```

## kvm启动命令示例
``` shell
sudo kvm -m 2048m -smp 2 \
-drive file=coreos-etcd.img,if=virtio,cache=none,aio=native \
-fsdev local,id=conf,security_model=none,readonly,path=~/myconfig \
-device virtio-9p-pci,fsdev=conf,mount_tag=config-2 \
-net nic,vlan=0,model=virtio \
-net tap,vlan=0,vhost=on,script=/etc/openvswitch/ovs-ifup,downscript=/etc/openvswitch/ovs-ifdown \
-nographic > /dev/null 2>&1 &
```

## 日志诊断
CoreOS的日志查看非常简单, 他提供了一个统一的工具: journalctl[fn:3], 通过这个工具, 你能查看单个节点的日志信息.

读整个日志, 不需要加任何参数, 直接执行即可. 查看某个服务的日志, 简单通过如下命令:

: $ journalctl -u apache.service

读启动日志, 可以查看最晚的启动日志, 方便诊断启动错误(cloud config出现问题可以清楚看到):

: journalctl --boot

你也能用fleetctl读取fleet控制的服务:

: $ fleetctl --tunnel 10.10.10.10 journal apache.service

当然也能tail ,直接在命令后面加-f参数即可

## 一些问题
1. Cloud config容易由于网络问题, 导致配置失败, 估计好多人都碰到了[fn:4].

# Footnotes

[fn:1] [[https://coreos.com/docs/#running-coreos][CoreOS support platforms]]

[fn:2] [[https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/][Cloud Config]]

[fn:3] [[https://coreos.com/docs/cluster-management/debugging/reading-the-system-log/][Reading the System Log]]

[fn:4] [[https://github.com/coreos/coreos-cloudinit/issues/205][CoreOS Cloud init #205]]
