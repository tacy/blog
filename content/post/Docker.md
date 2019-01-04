---
title: "Docker使用笔记"
date: 2014-04-04
lastmod: 2014-08-26
draft: false
tags: ["tech", "virtualization", "linux", "container"]
categories: ["tech"]
description: "Docker是对LXC的一个抽象，依赖于LXC，相比KVM这种Hypervisor，LXC要轻量级很多，但是LXC很难打包分发，Docker就是解决这个问题，当然还提供了一系列的辅助功能，本文是自己在使用Docker使用过程中的一些记录"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Docker[fn:1]
Docker是对LXC的一个抽象，依赖于LXC，相比KVM这种Hypervisor，LXC要轻量级很多[fn:2]
，但是LXC很难打包分发，Docker就是解决这个问题，当然还提供了一系列的辅助功能，本
文是自己在使用Docker使用过程中的一些记录。

## 一个例子
先来感受一下，我想在我机器上跑一个Oracle，又不想跑在本地。之前都是通过KVM，然后
建一个Ubuntu虚拟机，安装OracleXE版本。用起来也挺方便，但还是觉得有些不爽，一个是
占地方，另外就是耗资源，开销挺大，多开几个就受不了。

现在我们来看看Docker怎么玩，首先我们在docker仓库查找一下，看看是否有活雷锋：
``` shell
tacy@tacy:~$ sudo docker search oracle
NAME                              DESCRIPTION                                     STARS     OFFICIAL   TRUSTED
wnameless/oracle-xe-11g           SYS & SYSTEM password: oracle https://inde...   5                    [OK]
alexeiled/docker-oracle-xe-11g    This is a spin off from wnameless/docker-o...   2                    [OK]
haroon/docker-oracle-jdk7         Trusted Oracle JDK7 on Ubuntu:12.10             1                    [OK]
```
很幸运，不用你自己费劲弄了，直接pull下来吧：
: sudo docker pull wnameless/oracle-xe-11g
等待下载，时间比较长，接近1G的大小，完成之后来看看：
``` shell
tacy@tacy:~$ sudo docker images
REPOSITORY                TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
wnameless/oracle-xe-11g   latest              f8d224b82290        17 hours ago        2.389 GB
ubuntu                    13.10               9f676bd305a4        8 weeks ago         178 MB
```

接下来按照该Image的[[https://index.docker.io/u/wnameless/oracle-xe-11g/][文档]]操作，直接启动一个Container（我这里添加了-name参数）：
``` shell
tacy@tacy:~$ sudo docker run -name oraclexe -d -p 49160:22 -p 49161:1521 wnameless/oracle-xe-11g
d14110dc542540c4b83a44d8ca7b9d393a9aa4be7a60cd1ca757463bc12df2a9
tacy@tacy:~$ ps -ef|grep docker
root      1599  1588  0 09:58 ?        00:02:43 /usr/bin/docker -d
root     14977  1599  0 16:10 ?        00:00:00 lxc-start -n d14110dc542540c4b83a44d8ca7b9d393a9aa4be7a60cd1ca757463bc12df2a9 -f /var/lib/docker/containers/d14110dc542540c4b83a44d8ca7b9d393a9aa4be7a60cd1ca757463bc12df2a9/config.lxc -- /.dockerinit -g 172.17.42.1 -i 172.17.0.2/16 -mtu 1500 -- /bin/sh -c sed -i -E "s/HOST = [^)]+/HOST = $HOSTNAME/g" /u01/app/oracle/product/11.2.0/xe/network/admin/listener.ora; service oracle-xe start; /usr/sbin/sshd -D
```
秒级启动，数据库连接信息：
: hostname: localhost port: 49161 sid: xe username: system password: oracle
sys用户密码是oracle，可以通过ssh连接到container，root密码是admin：
``` shell
tacy@tacy:~$ ssh root@localhost -p 49160
root@localhost's password:
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.11.0-15-generic x86_64)

# Documentation:  https://help.ubuntu.com/
root@d14110dc5425:~#
```

连接数据库:
``` shell
root@d14110dc5425:~# su - oracle
oracle@d14110dc5425:~$ sqlplus '/as sysdba'

SQL*Plus: Release 11.2.0.2.0 Production on Fri Apr 4 08:19:36 2014

Copyright (c) 1982, 2011, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production

SQL>
```

停止container：
: tacy@tacy:~$ sudo docker stop oraclexe
或者：
``` shell
tacy@tacy:~$ sudo docker ps
CONTAINER ID        IMAGE                            COMMAND                CREATED             STATUS              PORTS                                                      NAMES
d14110dc5425        wnameless/oracle-xe-11g:latest   /bin/sh -c sed -i -E   11 minutes ago      Up 11 minutes       0.0.0.0:49160->22/tcp, 0.0.0.0:49161->1521/tcp, 8080/tcp   oraclexe
tacy@tacy:~$ sudo docker stop d14110dc5425
d14110dc5425
tacy@tacy:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
tacy@tacy:~$
```

重新启动该容器：
: tacy@tacy:~$ sudo docker start oraclexe

是不是很酷，还不赶紧去试试。

## 运行一个DB2 Container
上面是个简单的例子，让你先对Docker有个认识，现在我们来自己配置一个Container，我
们一步一步来。

假设说你希望有一个跑DB2 Express-C（DB2的免费版本）的Container，搜索一下，仓库里
没有，只能自己配置一个。

首先我们需要从BASE启动一个Container，我们这里的测试环境是Ubuntu12.04/Docker0.75
，首先我们下载BASE的Image：
: sudo docker pull ubuntu

接下来创建并运行一个Container[fn:7]：
``` shell
tacy@tacy:~$ sudo docker run -i -name db2exc -t ubuntu /bin/bash
root@4e476fda5fb5:/# uname -a
Linux 4e476fda5fb5 3.11.0-15-generic #23~precise1-Ubuntu SMP Tue Dec 10 16:39:48 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
```
Kernel版本和宿主机一致。

DB2 Express-C在Ubuntu源里面提供了，这点比Oracle XE方便，你只需要apt安装就行,我
们先来找找：
: sudo apt-cache search db2exc
显示没有这个包，由于db2exc属于Canonical Partner软件，查看一下apt源配置文件：
``` shell
root@4e476fda5fb5:/etc/apt# pwd
/etc/apt
root@4e476fda5fb5:/etc/apt# cat sources.list
deb http://archive.ubuntu.com/ubuntu precise main universe
deb http://archive.ubuntu.com/ubuntu precise-updates main universe
deb http://archive.ubuntu.com/ubuntu precise-security main universe
```
源没有包括Partner源，我们从宿主机拷贝配置文件过来（简单编辑一下文件也行），但是
找了一圈，没发现什么好办法直接从Host拷贝文件到Container，不行咱们先装SSH，通过
scp来：
``` shell
root@4e476fda5fb5:/etc/apt# apt-get install ssh
root@4e476fda5fb5:/etc/apt# scp tacy@172.17.42.1:/etc/apt/sources.list .
tacy@172.17.42.1's password:
sources.list                                                                                                    100% 2278     2.2KB/s   00:00
root@4e476fda5fb5:/etc/apt# apt-get update
root@4e476fda5fb5:/etc/apt# apt-cache search db2exc
db2exc - DB2 Express-C 9.7 Fix Pack 5 for Linux
```

安装一下：
: apt-get install db2exc

完成之后，切换到db2inst1用户就可以创建数据库了（缺省没有建库），到这里貌似问题
就解决了？事情还没完呢，现在我希望能自动启动SSH服务和DB2，并且可以从外面管理数
据库和访问它。

尝试启动一下SSH服务：
``` shell
root@4e476fda5fb5:/usr/sbin# service ssh start
root@4e476fda5fb5:/usr/sbin# ps -ef|grep ssh
root@4e476fda5fb5:/usr/sbin#
```
没反应，其实很正常，Container并没有启动upstart服务，为了保证Container的轻量级，
当你创建一个Container的时候，它只会启动你命令行指定的命令（一般在dockerfile），
我们的Container启动的是/bin/bash，所以如果你要启动ssh，可以通过下面方式启动：
``` shell
root@4e476fda5fb5:/usr/sbin# /usr/sbin/sshd -D
Missing privilege separation directory: /var/run/sshd
root@4e476fda5fb5:/usr/sbin# mkdir /var/run/sshd
root@4e476fda5fb5:/usr/sbin# /usr/sbin/sshd -D &
[1] 14
root@4e476fda5fb5:/usr/sbin# ps -ef|grep sshd
root        14     1  0 06:36 ?        00:00:00 /usr/sbin/sshd -D
root        16     1  0 06:36 ?        00:00:00 grep sshd
```

能正常启动了，如果你想从外部连接进去，尝试一下：
``` shell
tacy@tacy:/var/cache/apt/archives$ ssh root@172.17.0.26
ssh: connect to host 172.17.0.26 port 22: Connection refused
```

无法连接，你需要先把端口映射到Host上，另外你也不知道root密码，我们先设置一下密
码，接下来你希望能修改启动参数，但是很不幸，Container启动之后，无法变更命令，你
需要新建一个Container。[fn:3]

由于你需要基于当前Container的配置新建一个，但是Container只能从Image创建，因此你
需要把当前的工作提交一下，生成一个新的Image，提交Image：
``` shell
tacy@tacy:/etc/init.d$ sudo docker commit db2exc tacylee/db2exc
250b78860e291ebd71948bbb596f7dfc1a7d61563184a433bb44a8f2363079cb
tacy@tacy:/etc/init.d$ sudo docker images
REPOSITORY                TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
tacylee/db2exc            latest              250b78860e29        24 seconds ago      1.349 GB
wnameless/oracle-xe-11g   latest              f8d224b82290        5 days ago          2.389 GB
```

基于我们提交的Image运行一个Container：
``` shell
tacy@tacy:/etc/init.d$ sudo docker run -d -p 49170:22 -p 49171:50000 -name tt tacylee/db2exc /bin/bash -c '/etc/init.d/db2exc start && /usr/sbin/sshd -D'
c71ac7f85e167850552809ec12a12e50e064d30dddfeb4d71302d16d0ac34502
```

这个命令行有几个地方要注意，首先我们通过‘-d’参数让我们的Container运行在detached
模式（也就是后台模式），其次我们通过‘-p’参数把Container端口22和50000映射到Host，
另外就是CMD，这地方比较晕，弄了好久才明白[fn:4][fn:5]。关键点是，每个Container必须
有一个前台服务在运行，如果这个前台服务停止了，Container就停止了[fn:6]，例如：
``` shell
tacy@tacy:~$ sudo docker run -d ubuntu echo 'helloworld'
3088fce883ece6e04282ea965af84ddd4165b67873811e949b543f8930aebe82
tacy@tacy:~$ sudo docker ps -a
CONTAINER ID        IMAGE                            COMMAND                CREATED             STATUS              PORTS                                                      NAMES
3088fce883ec        ubuntu:12.04                     echo helloworld        20 seconds ago      Exit 0
```
可以看到，这个Container已经停止了，因为他没有前台进程在跑了，所以如果你需要你的
Container提供服务，你需要一个前台进程，很多时候我们用sshd来实现。接下来是要让我
们的Container启动DB2，使用命令组合实现即可。注意不能让sshd放在前面执行，因为他不
会结束，所以在它后面的命令无法执行。

到这里，我们貌似已经已经有了一个跑DB2的Container了。但是，这个世界总是没那么美好
，我们先看看Container的启动日志：
``` shell
tacy@tacy:~$ sudo docker logs tt
tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
tput: No value for $TERM and no -T specified
error: permission denied on key 'kernel.msgmni'
error: permission denied on key 'kernel.msgmnb'
error: permission denied on key 'kernel.msgmax'
error: permission denied on key 'fs.file-max'
# Starting DAS:			failed.

Message: SQL4401C  The DB2 Administration Server encountered an error during startup.

# Instance db2inst1 ( db2c_db2inst1 ):	done.
```

进去看看/etc/init.d/db2exc脚本，发现它会去修改kernel参数，而Docker为了保证安全
，这些权限是被关闭的。简单做法是在Container里面配置好，然后别让脚本执行Kernel
参数的设置操作，另外如果你不想去配置，可以让你的Container运行在特权模式，只需在
创建Container的时候加上‘-privileged‘参数，这样Container就能执行任何权限指令了。

Kernel参数问题解决了之后，你通过SSH连接到Container，切换到db2inst1用户，尝试建
立数据库，很不幸，你会碰到下面错误：
``` shell
db2inst1@c71ac7f85e16:~$ db2 create db dd
SQL1042C  An unexpected system error occurred.  SQLSTATE=58004
```

查看DB2日志文件，你会发现如下错误信息：
``` shell
2014-04-11-11.44.10.392525+000 E100588E1093        LEVEL: Error (OS)
PID     : 369                  TID  : 140502687016704PROC : db2sysc
INSTANCE: db2inst1             NODE : 000          DB   : DD
EDUID   : 79                   EDUNAME: db2loggr (DD)
FUNCTION: DB2 UDB, oper system services, sqloopenp, probe:80
MESSAGE : ZRC=0x870F0002=-2029060094=SQLO_BPSE "Debug logic error detected"
          DIA8501C A buffer pool logic error has occurred.
CALLED  : OS, -, open                             OSERR: EINVAL (22)
DATA #1 : Codepath, 8 bytes
4:11:15:20:22:37
DATA #2 : File name, 63 bytes
/home/db2inst1/db2inst1/NODE0000/SQL00001/SQLOGDIR/S0000000.LOG
DATA #3 : SQO Open File Options, PD_TYPE_SQO_FILE_OPEN_OPTIONS, 4 bytes
SQLO_REVISE, SQLO_READWRITE, SQLO_SHAREREAD, SQLO_WRITETHRU, SQLO_SECTOR_ALIGNED, SQLO_NO_THREAD_LEVEL_FILE_LOCK
DATA #4 : Hex integer, 4 bytes
0x00000180
DATA #5 : signed integer, 4 bytes
0
DATA #6 : signed integer, 4 bytes
16384
DATA #7 : String, 105 bytes
Search for ossError*Analysis probe point after this log entry for further
```

这个问题折腾了我两天，对Linux编程不了解的恶果。导致该问题的原因是Docker在mount
rootfs的时候，用的aufs，但是不支持O_DIRECT[fn:8][fn:9][fn:10][fn:12]，而DB2对有
些文件的操作用到O_DIRECT，比如日志，这就导致了该问题。上面的链接中提到0.7版本已
经mount为ext4，并且支持O_DIRECT，但实际情况看貌似没有。解决办法上面的链接里面都
提到了，我这里采用的办法是直接bind-mount了Host的一个目录到Container，该目录位于
ext4文件系统，并且支持O_DIRECT，最终的Container创建命令如下：
: sudo docker run -d -p 49170:22 -p 49171:50000 -v /home/tacy/Docker/db2db:/home/db2inst1/db2db -name db2exc -privileged tacylee/db2exc /bin/bash -c '/etc/init.d/db2exc start && /usr/sbin/sshd -D'
然后把数据库创建在/home/db2inst1/db2db目录中即可。

如果你不想bind-mount Host的目录到Container（毕竟很不灵活），也可以考虑替换Docker
的后台存储为Devicemapper[fn:11]。

对于这类问题的诊断方法可以通过strace来实现，例如上面的例子：从DB2的日志里面我们
知道是db2sysc需要执行系统的OPEN操作出现问题，你可以通过stace来捕获具体的错误。

我们先获取db2sysc的进程ID：
``` shell
root@314f5d6aa350:~# ps -ef|grep db2sysc
db2inst1   369   367  0 11:30 ?        00:00:23 db2sysc
root      2894  2881  0 15:12 pts/1    00:00:00 grep --color=auto db2sysc
```

然后用下面命令trace db2sysc的所有open操作：
``` shell
root@314f5d6aa350:~# strace -Ff -e open -o trace.out -p 369
Process 369 attached with 36 threads - interrupt to quit
```

在另一个终端执行数据库创建操作，完成之后，分析trace.out文件：
``` shell
2973  open("/home/db2inst1/sqllib/profile.env", O_RDONLY) = -1 ENOENT (No such file or directory)
2973  open("/etc/passwd", O_RDONLY|O_CLOEXEC) = 42
2973  open("/var/db2/global.reg", O_RDONLY|O_CREAT, 0644) = 42
2973  open("/home/db2inst1/db2inst1/NODE0000/SQL00001/SQLOGCTL.LFH.1", O_RDWR|O_DSYNC) = 42
2973  open("/home/db2inst1/db2inst1/NODE0000/SQL00001/SQLOGCTL.LFH.2", O_RDWR|O_DSYNC) = 43
2973  open("/proc/mounts", O_RDONLY)    = 45
2973  open("/home/db2inst1/db2inst1/NODE0000/SQL00001/SQLOGDIR/S0000000.LOG", O_RDWR|O_DSYNC|O_DIRECT) = -1 EINVAL (Invalid argument)
```

看截取部分的最后一行，提示open操作的参数错误，基本你就能判断是O_DIRECT无法执行
导致，这属于换个思路解决问题，也很有效果。

到这里，一个能跑DB2 Express-C的Container才算完成了，唯一的遗憾就是挂着个外部目
录，不方便迁移，等待Docker解决吧。

## Docker仓库
前面我们提到Docker的仓库，里面有各种Image提供，你也可以把你的Image提交到仓库，供
别人使用，你需要先去[[https://index.docker.io/][Docker Index]]注册一个帐号，然后通过docker login命令登入，再通
过docker push即可，具体我也没有试过，等我把DB2的Image弄好了，或许去提交一个玩玩。

## TIPS
### 进入正在运行的容器
docker处于运行状态的容器如果在交互模式，你能通过screen/tmux等方式实现新开窗口运
行，完成多任务操作，比如debug等。

对于运行在前台的任务，比如java应用，如果你想进入该容器终端，有几种方式：
- 启动监听服务
比如ssh，你可以通过ssh登入容器，但是该方式的缺点就是太重，管理复杂
- 使用lxc-attach
: docker ps --no-trunc
获取到container id，然后lxc-attach上去。不过0.9版本之后需要修改配置[fn:14]
- 使用nsenter
使用大神jpetazzo[fn:13]开发的nsenter，很方便

### 监控docker cli和server的交互
docker进程启动的时候创建了unix domain socket文件在/var/run/docker.sock,你可以通
过socat/nc之类的工具连接上去,和docker服务进程交互,docker cli也是通过该socket和服
务进程进行交互。

如果你想写一个自己的客户端，除了参考REST API之外，你也可以通过监控docker cli和
docker服务进程之间的交互来学习。

要实现这个，你只需要通过socat实现一个docker socket的代理：
: sudo socat -t100 -v UNIX-LISTEN:/tmp/docker.sock,mode=777,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock

然后在另一个终端像如下操作cli：
: docker -H unix:///tmp/docker.sock ps

你就可以在socat窗口看到交互的内容，你将看到例如下面的输出：
``` shell
> 2014/09/05 14:04:45.169982  length=130 from=0 to=129
GET /v1.13/containers/json?all=1 HTTP/1.1\r
Host: /tmp/proxysocket.sock\r
User-Agent: Docker-Client/1.1.0\r
Accept-Encoding: gzip\r
\r
< 2014/09/05 14:04:45.171377  length=991 from=0 to=990
HTTP/1.1 200 OK\r
Content-Type: application/json\r
Date: Fri, 05 Sep 2014 06:04:45 GMT\r
Content-Length: 882\r
\r
[{"Command":"/bin/bash","Created":1405345176,"Id":"044f8cb3e0d6522ec36e09
c3df0e7ff4bbf2d963d67be6d7faade6c88b5cbbb8","Image":"tianon/centos:5.8","
Names":["/sad_turing"],"Ports":[],"Status":"Exited (1) 6 weeks ago"}
,{"Command":"/bin/bash -c '/etc/init.d/db2exc start \\u0026\\u0026 /usr/s
bin/sshd -D'","Created":1397215845,"Id":"314f5d6aa350976b8c9071958ee0c8fd
0e2c700b8b348410be50e1ba77db4a07","Image":"tacylee/db2exc:latest","Names"
:["/db2exc"],"Ports":[],"Status":"Exited (137) 4 months ago"}
,{"Command":"\\"/bin/sh -c 'sed -i -E \\"s/HOST = [^)]+/HOST = $HOSTNAME/
g\\" /u01/app/oracle/product/11.2.0/xe/network/admin/listener.ora; servic
e oracle-xe start; /usr/sbin/sshd -D'\\"","Created":1397014650,"Id":"1379
9635f5ee898e38c0f300475a1eaf3ad3da6e18fc8ea978f54176a96d44c9","Image":"wn
ameless/oracle-xe-11g:latest","Names":["/oraclexe"],"Ports":[],"Status":"
Exited (-1) 6 weeks ago"}
]
```

你也可以建立tcp代理，这样远程客户端也能连接上来：
: sudo socat -t100 -v tcp-listen:5050,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock

现在你可以通过curl发起请求：
: curl http://localhost:5050/v1.13/containers/json?all=1

效果和unix domain socket一样，当然你也可以通过docker cli请求：
: docker -H tcp://localhost:5050 ps -a

另外的方式就是把docker服务进程启动的时候指定tcp监听，然后通过sniffer方式去获取
交互内容。


### 关于层的限制
注意,docker image限制层高127,也就是说一个image的layer不能超过127,你在build image
的时候必须注意,run和add都会产生层,另外docker存在冗余文件问题,比如你在build文件
里面先做了一个删除,然后在下一个run里面有做了添加操作,这个文件就会存在两份,导致
image非常臃肿.

关于这个问题的一些参考issue[fn:15][fn:16][fn:17]

### 删除容器
: sudo docker rm -v <CONTAINER ID>
注意，需要带上"-v"参数，否则不会删除aufs层文件，造成垃圾数据。

### 删除一个本地image
删除命令如下：
: sudo docker rmi <IMAGE ID>

如果你碰到如下错误：
``` shell
tacy@tacy:~$ sudo docker rmi 8dbd9e392a96
Error: Conflict, 8dbd9e392a96 wasn't deleted
2014/04/04 14:07:55 Error: failed to remove one or more images
```

很可能是由于该image正在被使用，你可以通过下面命令查看和操作
``` shell
sudo docker ps
sudo docker stop <CONTAINER ID>
sudo docker rm <CONTAINER ID>
sudo docker rmi <IMAGE ID>
```

# docker upgrade
## network
升级docker之后,启动发现docker另外创建了一个qemu0(我系统本省已经有一个qemu0, 它直接把我的删掉重建了一个, ip配置也变了. 我尝试降版本, 发现启动不了, 抛无法创建网桥错误, 解决方法是删除/var/lib/docker/network, 然后重启docker服务, 参考https://github.com/docker/docker/issues/23630

## network namespace
[http://murat1985.github.io/kubernetes/cni/2016/05/14/netns-and-cni.html](Experiments with container networking: Part 1)
pid=`docker inspect -f '{{.State.Pid}}' $container_id`
ln -s /proc/$pid/ns/net /var/run/netns/$container_id
ip netns exec $container_id tcpdump -i any tcp port 8888 -nnn -v


# docker-hub automated build
记住, 在github上, 对应的reponsitory也要和docker绑定, 否则不会触发自动编译.
Within GitHub, a Docker integration appears in your repositories Settings > Webhooks & services page.
[https://docs.docker.com/docker-hub/builds/#understand-the-build-process](Configure automated builds on Docker Hub)


## command
save & export, save 保留层信息, export类似快照, 没有元信息, 包括cmd类似都没有


# dockerization
## kafka
All-in-one: https://github.com/spotify/docker-kafka

``` abap
docker run -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=172.18.1.0 --env ADVERTISED_PORT=9092 spotify/kafka
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

# Centos

[[http://wiki.centos.org/Cloud/Docker][Cloud/Doker - Centos Wiki]]

# Docker 重要的特性
- 支持版本管理
- docker hub 支持自动build image

# Security
Docker的安全还是需要完善，理论上VM的安全性高于Container，Container只要被escalate，就没法幸免，攻击面很大，而VM相对来说又多一层保护，毕竟escalate出去了，依然还有VMM保护，不会危及到整个host，比较所有和底层的交互都需要通过hypercall来实现，攻击面相对较小。

总的来说，要保证docker的安全，有几件事情需要注意：
- 首先尽量别别运行container在root下，docker目前没有支持user namespace，container里面的root权限和host下的root一致，很危险，而且目前user namespace也是出来不就[fn:22]，有待稳定。
- 如果你需要运行root权限应用，需要使用SELinux/Apparmor配合，控制container root权限。
- net=host不安全，root权限用户可以直接重启host，需要注意。
- 不要从不可信网站下载image
- 禁止不必要的权限，比如non-root升级到root问题可以通过移除文件的suid位
- 使用user namespace（参考docker issue #7906[fn:23]，支持user map

关于docker安全的一些参考：
- [[http://blog.docker.com/2013/08/containers-docker-how-secure-are-they/][Containers & Docker: How Secure Are They?]]
- [[https://news.ycombinator.com/item?id%3D6252182][Hacker News discuss - Containers and Docker: how secure are they?]]
- [[http://www.projectatomic.io/blog/][Project Atomic News (blog)]]
- [[http://opensource.com/business/14/7/docker-security-selinux][Are Docker containers really secure?]]
- [[https://fedorapeople.org/~dwalsh/SELinux/Presentations/DockerSecurity][Docker Security]]
- [[http://www.slideshare.net/jpetazzo/docker-linux-containers-lxc-and-security][Docker, Linux Containers (LXC), and security]]

# Storage backend
- aufs
  使用上注意，这个不能用在产品上，基于文件目录的层，不是COW方式。
- device mapper
  缺省用的是loop device，从性能考虑，需要用block device[fn:24]
-
# Migration
- CRIU[fn:25]
# Monitor
- cAdvisor

# Service Discovery
- etcd
- consul
  - Consul will use a load-balancing strategy similar to round-robin when it returns DNS answers
- self
- zookeeper

# Clusetr Management
- fleet
- mesos/marathon
- google omega
- yarn

- [[http://aws.amazon.com/blogs/aws/cloud-container-management/][ECS (EC2 Container Service)]]
- [[https://cloud.google.com/container-engine/][Google Container Engine]]
- [[https://wiki.openstack.org/wiki/Docker][Openstack Docker]]

# Docker registry
[[https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04][How To Set Up a Private Docker Registry]]

``` shell
$docker tag dockerfile/redis 172.16.11.1:5000/tacy/redis
$docker push 172.16.11.1:5000/tacy/redis
$docker pull 172.16.11.1:5000/tacy/redis
```

# Docker PaaS
- flynn
- deis

# Coreos
- etcd
- fleet
- docker

# PaaS
- marathon
- flynn
- deis
- zenoss control center

# Kubernetes
## pod
pod由同一host上的一组容器构成, 一般是一组紧耦合的进程. 这里不再docker container里面运行多个进程实现, 主要考虑是让docker container尽量轻量级, 不用负责进程管理, 更多的工作让k8完成.

一个pod是一个最小调度单元, 有自己的ip地址. 同一个pod内的容器共享网络空间.

## service

# IBM Research Report 2014 for Container and VM
IBM的一个关于容器和虚拟机的对比报告，主要是性能关注，包括CPU、内存、存储、网络
资源，测试结果显示容器的性能表现全面高于或等于虚拟机，对于IO敏感的应用，两个技
术都需要进行优化

隔离和资源控制是两个关键的需求对于多种负载类型的应用共享资源池，不能让他们相互
之间影响彼此，同时控制每个应用使用资源的多少，不至于出现资源争抢。

## 背景知识
首先了解虚拟化的难点（这里谈论的都是x86）。传统的x86系统都是基于裸硬件设计的，
也就是说OS缺省默认是自己拥有所有的底层硬件资源，CPU在设计的时候，都会把指令按照
安全级别分类，也就是通常说的protection ring[fn:18]，intel用2bit实现，也就是有
ring 0 ~ ring 3四个级别：ring 0权限最大，可以执行所有操作，ring 1 和ring 2 没有
权限操作硬件，ring 3权限最小。主流的x86操作系统都只使用了ring 0 和ring 3，ring 0
也被称为kernel mode，ring 3为user mode，kernel mode封装了一系列的system call，当
user mode需要做特权指令调用的时候，只能通过system call实现，产生ContextSwitch，
带来重的系统开销。

由于x86操作系统都直接占用了ring 0，对硬件设计没有考虑虚拟化的支持，那么虚拟化软件
厂商就各显神通，早期的虚拟化VMM的实现都是把VMM放到ring 0，guest os放在ring 1执行
，应用跑在ring 3，这样就带来一系列问题：

### 软件虚拟化问题
#### 特权指令调用
最早的vmware采用binary translation解决这个问题，在vmm层模拟所有的系统调用，当
guest os执行systemcall时，vmm捕获执行模拟操作，然后返回结果给guest os kernel，
继续返回给应用。软件实现方式带来很大开销，早期的虚拟化软件性能有很大问题，但是
他又一个好处就是对上层guest os透明，不需要对guest os进行任何修改。

XEN采用一种不同的方式：Paravitualization[fn:19]，也被称作半虚拟化技术，这种方式
的好处就是性能提升，但是问题就在于需要上层的guest os感知，也就是需要对guest os
就行修改，在芯片厂商的虚拟化支持没有出来之前，这个成为一种主流的方式，使得虚拟
化技术在服务器上运行（后期的vmware应该是个混合体）。

#### 内存管理问题（MMU）
由于cpu只有一个MMU，guest os上的内存寻址带来问题，需要从GVA-》GPA-》HVA-》HPA，
带来严重的性能开销，解决办法就是Shadow Page Table，vmm维护一个虚拟的TLB，直接
映射GVA-》HPA，但是这样的方式依然有很大开销，需要用软件方式额外维护一个频繁更
新的页表。

#### IO问题
早期的实现都是基于模拟方式，主流的QEMU实现了全虚拟的驱动设备，但是效率非常地下。
后期主流都采用Paravirtualization实现，但是需要在guest os上安装具体驱动。vmware
需要在guest os上安装vmtools就是这个原因。xen为了支持windows也开发了一套
GPLPV驱动。

### 硬件支持
x86芯片虚拟化自然也是不干落后，首先是vt-x（只说intel，AMD有相应技术）[fn:20]，
第一代的vt-x解决的是特权指令问题，引入了ring -1（也叫Root mode）,VMM跑在Root
mode，但是第一代的vt-x并没有大规模使用，大量的vmexit和vmenter以及太高的消耗（
几百到几千的cycles），导致性能比软件实现还差，只是简化了VMM开发。[fn:21]

第二代的vt-x优化了实现，并且引入了EMT，硬件上解决内存寻址问题，芯片内置了嵌套
页表，直接缓存GVA-》HPA映射，但是依然会带来一些问题，比如TLB Miss相比非虚拟化
方式开销巨大，芯片的解决办法是引入更大的TLB。虚拟软件厂商也通过采用大页的方式
来缓解这个问题。

对于IO问题的解决方案intel引入vt-d技术（也被称作IOMMU），直接passthrough硬件到
guest os，但是这种方式带来的问题是虚拟机无法迁移。

其他的解决方案比如PCIe的SR-IOV，直接支持pci卡的虚拟多设备绑定到guest os，都有
迁移问题。

IO这块通用的方式更多还是paravirtualization。

### KVM
kvm采用的是HVM方式，也就是必须硬件虚拟化支持（vt-x）。IO实现两种模式，一种是通
过QEMU emulator模拟驱动设备，性能很差，基本只做测试，另外一种就是paravirtual（
Virtio），损耗比较低，IO性能能达到95%左右，但是这只是测试，具体使用中会有各种
问题，比如IO对齐，缓存等。

kvm也支持guest os的vcpu、vram在线伸缩，这些在现有的IaaS中几乎没有看见使用，需
要上层软件支持。

另外kvm直接利用了linux的已有的资源调度和管理，每一个guest os就是一个process，这
简化了kvm实现，但是也带来的复杂度，比如内存，如果overcommitted，内存管理就无法
很好的swapout，guest os的性能就无法保证。所以很多IaaS也会提供cpu pin和资源不超
售，以及vRAM锁定到RAM的方式提高性能。

对于VM的安全，由于只能通过hypercalls和虚拟驱动设备和外界交互，而二者都需要通过
VMM，保证了VM的安全隔离，当然也不是说没有安全问题，vmm的安全漏洞也时有发现。

VM的隔离也带来了其他开销：系统内存，文件系统等，虚拟化厂商也在提供技术解决方案
，比如内存的ballooning，但是这些都会带来其他开销。

### linux container
linux container底层技术就是两个：kernel namespace和control groups。

linux实现了fs、pid、network、user、ipc和hostname的名称空间，可以实现隔离。例如
每个fs名称空间都有自己的根系统，mount表，表现的就是一个独立的文件系统。每个网络
名称空间能有自己的网卡，自己的路由表等。

cgroups







# 一些需要注意的问题：
## net=host安全问题
   [[https://github.com/docker/docker/issues/6401][Rebooting within docker container actually reboots the host #6401 ]]

# 好的文章
- [[http://stackoverflow.com/questions/18285212/how-to-scale-docker-containers-in-production][How to scale Docker containers in production]]
- [[http://www.centurylinklabs.com/caching-docker-images/?hvid%3D2WaPeN][Working With the Docker Image Cache]]

# Footnotes

[fn:1] [[http://docs.docker.io/en/latest/faq/][Docker FAQ]]

[fn:2] [[http://stackoverflow.com/questions/16047306/how-is-docker-io-different-from-a-normal-virtual-machine?rq%3D1][How is Docker.io different from a normal virtual machine?]]

[fn:3] [[https://github.com/dotcloud/docker/issues/1228][running a different command on an existing container]]

[fn:4] [[https://github.com/dotcloud/docker/issues/1826][After importing an image, run gives "no command specified]]"

[fn:5] [[https://groups.google.com/forum/#!topic/docker-user/yv2RAnaXvpk][Using sudo inside Docker image]]

[fn:6] [[http://blog.trifork.com/2014/03/11/using-supervisor-with-docker-to-manage-processes-supporting-image-inheritance/][Using Supervisor with Docker to manage processes]]

[fn:7] [[http://docs.docker.io/en/latest/reference/run/][Docker Run Reference]]

[fn:8] [[https://github.com/dotcloud/docker/issues/1916][docker build should support privileged operations]]

[fn:9] [[https://github.com/dotcloud/docker/issues/3433][User defined layer mount options]]

[fn:10] [[https://groups.google.com/forum/#!topic/docker-user/1pFhqlfbqQI][DB2 in container can't start unless I mount non-aufs volume from host]]

[fn:11] [[http://muehe.org/posts/switching-docker-from-aufs-to-devicemapper/][Switching Docker from aufs to devicemapper]]

[fn:12] [[http://webcache.googleusercontent.com/search?q%3Dcache:tp3o-xrNBycJ:permalink.gmane.org/gmane.comp.sysutils.docker.user/950%2B&cd%3D4&hl%3Den&ct%3Dclnk][Re: Oracle Database won't run in docker container]]

[fn:13] [[https://github.com/jpetazzo][jpetazzo blog]]

[fn:14] [[http://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/][Attaching to a container with Docker 0.9 and libcontainer]]

[fn:15] [[https://github.com/docker/docker/issues/332][flatten images - merge multiple layers into a single one]]

[fn:16] [[https://github.com/docker/docker/issues/2439][Dockerfiles should have a way to perform multiple build actions in one commit]]

[fn:17] [[https://github.com/docker/docker/pull/4232][Add docker squash command]]

[fn:18] [[http://en.wikipedia.org/wiki/Protection_ring][Protection ring]]

[fn:19] [[http://en.wikipedia.org/wiki/Paravirtualization][Paravirtualization]]

[fn:20] [[http://en.wikipedia.org/wiki/X86_virtualization][x86 virtualization]]

[fn:21] [[http://www.anandtech.com/print/2480/][Hardware Virtualization: the Nuts and Bolts]]

[fn:22] [[http://lwn.net/Articles/532593/][Namespaces in operation, part 5: User namespaces]]

[fn:23] [[https://github.com/docker/docker/issues/7906][Proposal: Support for user namespaces]]

[fn:24] [[http://jpetazzo.github.io/2014/01/29/docker-device-mapper-resize/][Resizing Docker containers with the Device Mapper plugin]]

[fn:25] [[http://en.wikipedia.org/wiki/CRIU][CRIU]]
