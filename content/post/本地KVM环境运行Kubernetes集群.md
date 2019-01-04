---
title: "本地KVM环境运行Kubernetes集群"
date: 2014-11-24
lastmod: 2014-11-24
draft: false
tags: ["tech", "virtualization", "kubernetes", "container"]
categories: ["tech"]
description: "Kubernetes作为目前热门的容器集群管理软件, 很多人都希望能一探究竟, 但是它的文档大部分都是运行在IaaS平台之上, 对于没有IaaS环境的人来说, 还是有一定门槛, 这篇文章的目的就是让你能在自己的机器上搭建一个Kubernetes集群"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

Kubernetes(后面简称为K8S)作为目前热门的容器集群管理软件, 很多人都希望能一探究竟, 但是它的文档大部分都是运行在IaaS平台之上, 对于没有IaaS环境的人来说, 还是有一定门槛, 这篇文章的目的就是让你能在自己的机器上搭建一个K8S集群.

这里我采用的方式Kelsey Hightower提供的: [[https://github.com/kelseyhightower/kubernetes-fleet-tutorial][Elastic Kubernetes cluster with fleet and flannel]], 建议你先看看链接里的项目, 有一个大概的了解.

主要用到的技术栈包括:
- KVM 底层虚拟化平台
- OpenVSwitch 虚拟交换机(取代bridge)
- CoreOS 轻量级的操作系统, 和Docker天然集成, 独特的升级机制(和Google Chrome一样[fn:1], 不是一个一个包升级, 而是Core的完整升级, 这样能保证整个环境的一致性)
- Fleet CoreOS管理集群管理工具, 就是一个配置管理工具, 当然只支持CoreOS, 类比的工具就是Chef/SaltStack一类的东西
- Flannel CoreOS提供的overlay网络软件, 为K8S提供网络服务
- Etcd  CoreOS提供的配置共享/服务发现的软件
- K8S Google开源的容器集群管理软件

要完成整个配置, 你需要有一台至少8G RAM的机器, 少点估计也行, 但是建议还是多点, 毕竟要完成整个测试, 你需要有至少4个虚拟机, 如果需要跑监控的话 ,4G是打不住的.

首先我们需要准备好的是我们的底层虚拟化环境, 其实没有想象的那么麻烦, 配置还是比较方便的.

# 虚拟化环境准备

我们这里测试机器用的是操作系统是Ubuntu 14.04, 最小安装即可, 配置好SSH, 如果你没有物理机, 也可以用虚拟机代替, 没啥影响, 保证虚拟机网络通就行.

安装虚拟化支持软件:

: apt-get install qemu-kvm openvswitch-switch

配置OpenVSwitch
``` shell

$cat /etc/network/interface
auto ovsbr0
iface ovsbr0 inet static
   address 172.16.11.1
   network 172.16.11.0
   netmask 255.255.255.0
   broadcast 172.16.11.255

$ovs-vsctl add-br ovsbr0

$cat /etc/openvswitch/ovs-ifup
#!/bin/sh
switch='ovsbr0'
/sbin/ifconfig $1 0.0.0.0 up
ovs-vsctl add-port ${switch} $1

$cat /etc/openvswitch/ovs-ifdown
#!/bin/sh
switch='ovsbr0'
/sbin/ifconfig $1 0.0.0.0 down
ovs-vsctl del-port ${switch} $1

```

添加一个bridge ,然后添加到OVS里面, 同时编写两个脚本, 后面KVM命令行启动虚拟机要用到.

完成之后, 配置机器路由转发功能, 首先打开转发功能, 编辑/etc/sysctl.conf, 打开ip_forward=1那一行, 同时让配置生效:
: sysctl -p

配置iptables:
``` shell
iptables -I FORWARD -j ACCEPT
iptables -t nat -A POSTROUTING -s 172.16.11.0/24 ! -d 172.168.11.0/24 -j MASQUERADE
```

对所有172.16.11.0/24网段出去的流量做NAT, 同时转发所有的包. 这里只是测试环境, 所以我打开了所有包的转发, 当然你也能细粒度控制包转发.

完成上面两步只后, 你还需要有一个dhcp server, 为你创建的虚拟机提供DHCP服务, 可以用dnsmasq这个软件来完成.
``` shell
$apt-get install dnsmasq-base

$cat resolv.dnsmasq.conf
# Google's nameservers, for example
nameserver 8.8.8.8
nameserver 8.8.4.4

$dnsmasq --bind-interfaces --listen-address 172.16.11.1 --dhcp-range 172.16.11.100,172.16.11.200 --resolv-file=/home/tacy/coreos/config/resolv.dnsmasq.conf --dhcp-boot=pxelinux.0,roo,172.16.11.2
```

这样, 一个基本的虚拟化环境就差不多了.


# 配置etcd
这里需要简单介绍一下CoreOS这个操作系统, 不像其他传统的Linux发行版本, 他没有包管理机制, 只包括核心的一些软件, 升级操作是原子的, 不存在不一致问题.

当运行系统检测到CoreOS有升级包的时候, 他会根据你选定的升级策略, 自动下载升级包更新, 但是他不会更新你当前正在运行的根分区(/usr被mount为只读), 因为他存在两个根分区, Active/Passive, 他升级的是Passive这个分区, 重启的时候, 切换Passive成为Active, 完成升级操作.

CoreOS和Docker无缝集成, 预安装Docker, 如果你需要在CoreOS里面跑其他诊断工具, 比如tcpdump, 他通过创建一个fedora container来完成[fn:2].

CoreOS采用Systemd做服务管理, Systemd也是目前热门的服务管理工具, 不远的将来会替换SysV和Upstart, 好处就是服务并行启动和好的依赖设计. 结合Cloud-Config, 解决CoreOS系统的预配置工作, 不依赖其他配置管理工具, 比如Chef等.

同时CoreOS提供Fleet管理工具来管理CoreOS集群, Fleet通过发布Systemd Unit到集群成员, 实现服务配置.

当然, CoreOS的整个体系中, 我们将要配置的Etcd无疑是最重要的一个, Etcd负责配置管理和服务发现, 是CoreOS的核心. K8S也用Etcd.

介绍完了CoreOS, 我们来看怎么配置一个运行Etcd service的CoreOS.

Kelsey Hightower已经帮你编写好了Cloud-Config文件, 你需要先Clone一下上面提到的项目.

: git clone https://github.com/kelseyhightower/kubernetes-fleet-tutorial.git

找到config目录下的etcd.yml文件, 根据你的实际情况, 修改里面的ip地址信息, 我测试环境用的是172.16.11.10, 做相应修改, 同时吧ssh_authorized_keys替换成你的的公钥. 另外需要特别注意的是, 这里的网卡名称需要修改一下, 不是eno16777736, 改成eth0.

然后我们还需要下载一个CoreOS的KVM image. 去 [[https://coreos.com/docs/running-coreos/platforms/qemu/][这里]] 下载:

: wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_qemu_image.img.bz2 -O - | bzcat > coreos_production_qemu_image.img

准备好之后, 我们就可以来启动一个etcd服务的CoreOS虚拟机了. 由于我们需要使用Cloud-Config文件启动, 你需要把配置文件放置到一个特定的位置, 我们先创建一个目录:
``` shell
$mkdir -p tmp/openstack/latest
$cp etcd.yml tmp/openstack/latest/user_data
```

也就是把我们修改好的etcd.yml文件更名为user_data, 放置到/tmp/openstack/lastest目录下, 目录名的后两层不能修改, tmp可以任意. 这样CoreOS才能找到配置文件.

现在可以启动etcd server了, 在启动之前, 我们还做最后一步工作, 用coreos_production_qemu_image.img做base image, 建立一个新的img文件来写入etcd server的修改:
: $qemu-img create -f qcow2 -b ../template/coreos.img etcd.img

这里我重命名coreos_production_qemu_image.img为coreos.img, 启动etcd server:
``` shell
$kvm -m 2048 -smp 2 \
-drive file=../images/etcd.img,if=virtio,cache=none,aio=native \
-fsdev local,id=conf,security_model=none,readonly,path=../config/etcd \
-device virtio-9p-pci,fsdev=conf,mount_tag=config-2 \
-net nic,vlan=0,model=virtio,macaddr=52:54:00:24:93:29\
-net tap,vlan=0,vhost=on,script=/etc/openvswitch/ovs-ifup,downscript=/etc/openvswitch/ovs-ifdown \
-nographic -serial file:../logs/etcd.log
```

我的目录结构如下, Cloud-Config文件统一放到config目录, images下面存放虚拟机镜像, 虚拟机启动日志存放到logs目录:
``` shell
|-coreos
   |---bin
   |---config
   |-----etcd
   |-------openstack
   |---------latest
   |-----node
   |-------openstack
   |---------latest
   |---example
   |-----guestbook
   |---images
   |---logs
   |---temp
   |---template
   |---tools
   |---units
```

好了, 现在你的etcd server就配置好了,  当然你还需要配置两个环境变量:
``` shell
$ export ETCDCTL_PEERS=http://172.16.11.10:4001
$ export FLEETCTL_ENDPOINT=http://172.16.11.10:4001
```

同时在etcd创建flannel网络节点:
: $etcdctl mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'

用fleetctl验证一下:
``` shell
tacy@momo1:~/coreos$ fleetctl list-machines
MACHINE         IP              METADATA
6a6e4c00...     172.16.11.10    role=etcd
```

现在你可以继续配置其他虚拟机节点了, 这些节点用来运行K8S


# 配置其他Node
同样你需要基于coreos.img创建相应的img, 修改node.yml配置文件, 修正etcd server地址, 同时注意, node.yml里面的K8S下载用的是Google的云存储空间, 国内访问不了, 而且版本也是老的, 建议自己下载最新版本, 放置到本地, 如果你没有相关服务, 也可以临时用下面命令应急:

: $python -m SimpleHTTPServer

配置完成之后验证一下:
``` shell
tacy@momo1:~/kubernetes/server/kubernetes/server/bin$ fleetctl list-machines
MACHINE         IP              METADATA
127d44a9...     172.16.11.165   role=kubernetes
6a6e4c00...     172.16.11.10    role=etcd
7b0826fb...     172.16.11.132   role=kubernetes
bb8edb08...     172.16.11.178   role=kubernetes
```

可以看到我这里创建了3个节点, 创建节点相关工作比较繁琐, 我写了一个脚本来完成:
``` shell
tacy@momo1:~/coreos/bin$ cat microcloud.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os
import random
import pickle
import argparse
import subprocess

parser = argparse.ArgumentParser(description='MicroCloud Ctl')

parser.add_argument('-i',
                    '--image',
                    help='kvm node name',
                    required = True)
parser.add_argument('-c',
                    '--clouddriver',
                    help='cloud config driver directory',
                    required = True)

args = parser.parse_args()
img = args.image
cdf = args.clouddriver

pypath = os.path.dirname(os.path.abspath(__file__))
node = img.split('/')[-1]
macaddrfile = '{0}/../config/macaddr.list'.format(pypath)
if not os.path.isfile(macaddrfile):
    with open(macaddrfile, 'wb+') as fd:
        pickle.dump({}, fd)
macs = {}
with open(macaddrfile, 'rb+') as fd:
    macs = pickle.load(fd)
mac = ''
if node in macs:
    mac = macs[node]
else:
    rands = [ 0x52, 0x54, 0x00,
              random.randint(0x00, 0x7f),
              random.randint(0x00, 0xff),
              random.randint(0x00, 0xff) ]
    mac = ':'.join(map(lambda x: "%02x" % x, rands))
    macs[node] = mac
    with open(macaddrfile, 'wb+') as fd:
        pickle.dump(macs, fd)

args = ['qemu-system-x86_64', '-enable-kvm',
        '-m', '2048',
        '-smp', '2',
        '-drive', 'file={0},if=virtio,cache=none,aio=native'.format(img),
        '-fsdev', 'local,id=conf,security_model=none,readonly,path={0}'.format(cdf),
        '-device', 'virtio-9p-pci,fsdev=conf,mount_tag=config-2',
        '-net', 'nic,vlan=0,model=virtio,macaddr={0}'.format(mac),
        '-net', 'tap,vlan=0,vhost=on,script=/etc/openvswitch/ovs-ifup,downscript=/etc/openvswitch/ovs-ifdown',
        '-nographic',
        '-serial', 'file:{0}/../logs/{1}.log'.format(pypath, node)]

#os.execv('/usr/bin/qemu-system-x86_64', args)
process = subprocess.Popen(['/usr/bin/qemu-system-x86_64'] + args[1:],
                 stdout=subprocess.PIPE, stderr=subprocess.PIPE)
#stdout, stderr = process.communicate()
#print stdout, stderr
print process.returncode
```

有了这个脚本之后, 新建一个节点只需要创建一下img, 然后运行这个脚本就行了.


# 通过fleet部署K8S
CoreOS部署好之后, 你现在可以用fleet完成K8S的部署, 使用Kelsey Hightower提供的units, 直接用下面命令完成部署:
``` shell
$ fleetctl start kube-proxy.service
$ fleetctl start kube-kubelet.service
$ fleetctl start kube-apiserver.service
$ fleetctl start kube-scheduler.service
$ fleetctl start kube-controller-manager.service
```

同样的问题, 这里下载地址都是Google存储, 国内无法访问, 而且版本也是老的, 建议自己下载最新版本和配置下载地址.

完成之后, 你就有了一个本地运行的K8S集群.

需要特别注意的是, 由于K8S是一个快速变化的项目, 很多信息容易过期, 建议去关注项目的Issue列表, 由于0.5版本的K8S里面的服务重新设计, Kelsey Hightower文档中相关信息就不对, 注意看CoreOS的日志.

现在可以去部署Guestbook这个例子了.


# 部署Guestbook
直接下载最新发布的K8S版本, 解压缩之后里面有example/guestbook目录, 这就是我们需要部署的例子, guestbook部署包括一个redis master, 2个redis slave和3个web节点.

部署起来也和很简单, 直接通过kubectl工具即可完成:

``` shell
kubectl create -f redis-master.json
kubectl create -f redis-master-service.json
kubectl create -f redis-slave-controller.json
kubectl create -f redis-slave-service.json
kubectl create -f frontend-controller.json
```

看看结果:
``` shell
tacy@momo1:~$ kubectl get pods
NAME                                   IMAGE(S)                   HOST                LABELS                                       STATUS
8f85c3e8-7481-11e4-8c27-52540015fd4a   brendanburns/redis-slave   172.16.11.132/      name=redisslave,uses=redis-master            Running
8f85ff95-7481-11e4-8c27-52540015fd4a   brendanburns/redis-slave   172.16.11.165/      name=redisslave,uses=redis-master            Running
4229efd3-751b-11e4-8c27-52540015fd4a   brendanburns/php-redis     172.16.11.132/      name=frontend,uses=redisslave,redis-master   Running
422ac57d-751b-11e4-8c27-52540015fd4a   brendanburns/php-redis     172.16.11.178/      name=frontend,uses=redisslave,redis-master   Running
422bd487-751b-11e4-8c27-52540015fd4a   brendanburns/php-redis     172.16.11.165/      name=frontend,uses=redisslave,redis-master   Running
redis-master                           dockerfile/redis           172.16.11.165/      name=redis-master                            Running
```

这样就把guestbook部署完成了. 你在宿主机上可以直接通过80端口访问


# 部署监控
K8S的监控可以用cAdvisor + heapster + Influxdb + Grafana来实现. 下面我们来说说怎么去部署这样一个方案

首先需要做的是在每一个minions上运行cAdvisor作为一个Pod, 可以通过[[https://github.com/GoogleCloudPlatform/kubernetes/blob/master/cluster/saltbase/salt/cadvisor/cadvisor.manifest#L1][static manifest file]]来实现, 我们这里需要修改一下node节点的


# K8S存在的问题
## 服务对外暴露问题
对于服务的对外暴露问题, service提供的是一个虚的地址, 这对于内部路由没有问题(kube-proxy+iptables), 但是对于需要对外暴露的服务, 比如guestbook里面的web服务, 就没有很好的办法了(在GCE上是有办法解决的, 会判断service的createExternalLoadBalancer属性, 建立LB)

相关Issue: [[https://github.com/GoogleCloudPlatform/kubernetes/issues/1161][#1161]]

## 服务注入问题
需要解决Cluster内部Pod依赖外部服务问题, 比如集群间, 不同命名空间, 数据库等,

相关Issue: [[https://github.com/GoogleCloudPlatform/kubernetes/issues/1542][#1542]], [[https://github.com/GoogleCloudPlatform/kubernetes/issues/260][#260]], [[https://github.com/GoogleCloudPlatform/kubernetes/issues/2358][#2358]]

## 持久化数据问题

相关Issue: [[https://github.com/GoogleCloudPlatform/kubernetes/pull/1515][#1515]]


# 部署中碰到的问题
## etcd
etcd 0.5之前的版本使用的go-raft实现有问题, 会导致etcd起不来. 0.5版本之后, etcd实现了自己版本的raft, 解决了该问题, 建议用0.5之后的版本.
## fleetctl
fleetctl确实非常方便, 但是也有一些问题, 参考这里[fn:3][fn:4], 对于服务的管理还有bug, 期待更稳定
## kube-register
老版本有bug, 会有too much open files问题, 需要自己编译



# Footnotes

[fn:1] [[https://code.google.com/p/omaha/][Google Omaha]]

[fn:2] [[https://coreos.com/docs/cluster-management/debugging/install-debugging-tools/][Install Debugging Tools]]

[fn:3] [[https://github.com/coreos/fleet/issues/914][How to update unit?]]

[fn:4] [[https://github.com/coreos/fleet/pull/866][fleet agent does not compare contents of units in reconciler]]
