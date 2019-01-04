---
title: "systemtap notes"
date: 2014-06-10
lastmod: 2014-06-10
draft: false
tags: ["tech", "linux", "performance", "trace", "tuning", "tools"]
categories: ["tech"]
description: "systemtap是redhat开发的一个分析工具，功能非常强大，这里记录一点自己在使用上碰到的一些问题"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Systemtap
systemtap是红帽开发的一款分析工具，如果你需要使用的话，最好在redhat的系统上，在
Ubuntu上兼容性不好，坑非常多，Ubuntu缺省提供的Kernel无法正常运行Systemtap，只能
跑跑HelloWorld，带的那些脚本都无法跑通，没法用。只能自己重新编译kernel，修改一些
CFLAGS参数。

我电脑使用的是ubuntu 12.04，kernel用的是trusy版本的3.13，Systemtap缺省版本是1.6
，没法用。

## 使用前提
systemtap依赖dbgsym包，你必须安装kernel的dbgsym包，这个包在ubuntu的ddeb源提供[fn:1]，
系统缺省不包括，需要自己添加：
``` shell
tacy@tacy:~$echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-security main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list

tacy@tacy:~$sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 428D7C01

tacy@tacy:~$apt-get update
```

然后你就能看到很多dbgsym包，先安装对应kernel的
``` shell
tacy@tacy:~$ uname -a
Linux tacy 3.13.0-27-generic #50~precise1-Ubuntu SMP Fri May 16 20:47:56 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
tacy@tacy:~$ apt-cache policy linux-image-3.13.0-27-generic
linux-image-3.13.0-27-generic:
  已安装：  3.13.0-27.50~precise1
  候选软件包：3.13.0-27.50~precise1
  版本列表：
### 3.13.0-27.50~precise1 0
        500 http://mirrors.ustc.edu.cn/ubuntu/ precise-updates/main amd64 Packages
        500 http://mirrors.ustc.edu.cn/ubuntu/ precise-security/main amd64 Packages
        100 /var/lib/dpkg/status
tacy@tacy:~$ sudo apt-get install linux-image-3.13.0-27-generic-dbgsym=3.13.0-27.50~precise1
```

当然上面的前提是ubuntu提供的kernel没问题的前提下，如果你想在ubuntu下跑systemtap
可没那么轻松，必须自己编译kernel和dbgsym！

## 编译
### kernel编译
导致ubuntu提供的kernel不能运行的systemtap的根本问题还是gcc，gcc的一些bug导致
产生了错误的debuginfo，无法正确定位变量[fn:3]，产生如下错误信息：
``` shell
semantic error: not accessible at this address [man error::dwarf] (0xffffffff811b42b0, dieoffset: 0x177422e): identifier '$file' at /usr/share/systemtap/tapset/linux/vfs.stp:855:9
```

当然systemtap对此问题进行了修正，但是它需要有gcc编译时使用的编译参数才行。gcc提
供了一个参数来记录编译时参数，但是ubuntu编译kernel的时候并没有带上这个参数，导
致systemtap无法正确定位参数的位置，另外gcc的这个参数只有在4.7以上版本才有，而
12.04缺省带的gcc版本是4.6！

所以基于上面这些问题，我们需要做两件事情，首先是升级gcc版本到4.7，接下来是编译
kernel，同时加上gcc参数来记录编译项。

升级gcc我就不详细介绍了，直接参考VK的文章[fn:4]。

编译kernel最关键的是修改gcc编译参数，这个问题搞了好几天，没办法，对kernel还是不
熟。

首先是你需要知道ubuntu下编译kernel的基本流程[fn:5]。参考引用，我们先获取代码：

: apt-get source linux-image-`uname -r`

接下来你需要进入源码目录，修改最上层目录下的Makefile，我的目录如下：

: /home/tacy/Downloads/linux-lts-trusty-3.13.0/Makefile

修改内容如下：
``` shell
HOSTCFLAGS   = -Wall -Wmissing-prototypes -Wstrict-prototypes -grecord-gcc-switches -O2 -fno-omit-frame-pointer
HOSTCXXFLAGS = -O2 -fno-omit-frame-pointer -grecord-gcc-switches
。。。
KBUILD_CFLAGS   += -O2 -grecord-gcc-switches
```

主要是添加‘-grecord-gcc-switches‘和’-fno-omit-frame-pointer‘参数。

接下来就是编译了，如果你需要修改编译模块，直接用下面命令：

: fakeroot debian/rules editconfigs
: fakeroot debian/rules updateconfigs

注意下面项需要打开[fn:6]
``` shell
# for perf_events:
CONFIG_PERF_EVENTS=y
# for stack traces:
CONFIG_FRAME_POINTER=y
# kernel symbols:
CONFIG_KALLSYMS=y
# tracepoints:
CONFIG_TRACEPOINTS=y
# kernel function trace:
CONFIG_FTRACE=y
# kernel-level dynamic tracing:
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENTS=y
# user-level dynamic tracing:
CONFIG_UPROBES=y
CONFIG_UPROBE_EVENTS=y
# full kernel debug info:
CONFIG_DEBUG_INFO=y
# kernel lock tracing:
CONFIG_LOCKDEP=y
# kernel lock tracing:
CONFIG_LOCK_STAT=y
# kernel dynamic tracepoint variables:
CONFIG_DEBUG_INFO=y
```

编译：

: fakeroot debian/rules binary-generic skipdbg=false

然后安装编译包就行了，需要注意的时候编译过程会会占用16G左右的空间，并且编译过程
非常漫长，在我机器上跑了将近一个半小时，考验耐性～

### systemtap
没啥特殊的，注意elfutils版本就行，缺省的0.152好像没问题。我编译了systemtap 2.5版
本，可以用，直接下载的源码包编译就行。

## 使用
看看使用效果：

``` shell
tacy@tacy:/usr/local/share/doc/systemtap/examples/io$ sudo stap disktop.stp

Fri Jul  4 15:14:07 2014 , Average:  50Kb/sec, Read:     219Kb, Write:     32Kb

     UID      PID     PPID                       CMD   DEVICE    T        BYTES
    1000     2496     1861                    oracle     sda5    R       131072
    1000     2832        1           unity-panel-ser     sda5    R        76217
    1000     2496     1861                    oracle     sda5    W        32768
    1000     2834        1               hud-service     dm-1    R        16192
    1000     2689        1                   dropbox     sda5    R          894

Fri Jul  4 15:14:12 2014 , Average:  42Kb/sec, Read:     192Kb, Write:     18Kb

     UID      PID     PPID                       CMD   DEVICE    T        BYTES
    1000     2496     1861                    oracle     sda5    R       131072
    1000     8893     1861                    oracle     sda5    R        65944
    1000     2496     1861                    oracle     sda5    W        16384
    1000     5157        1           Chrome_CacheThr     dm-1    W         2033
    1000     2953        1            gnome-terminal     sda5    W         1025
    1000     5157        1           Chrome_CacheThr     dm-1    R           36
```

好了，ubuntu上也能systemtap啦～

## systemtap & byteman

另外需要注意的是systemtap可以配合byteman使用，但是～，基本上别抱啥希望，我尝试
了一下，问题一堆，能正常编译，无法使用，当然你也可以试试，一些需要注意的点：

仔细看systemtap源代码中java目录下的readme，这里有好几个坑：

首先是编译的时候要指定--with-java选项，否则不支持。

另一个是BYTEMAN_HOME，你从jboss网站上下载下来的byteman解压之后，jar包在lib目录下，
但是stapbm脚本直接忽略了lib目录，也就是说你找不到jar包，没有任何提示，无法成功。

其次是libHelperSDT_amd64.so和HelperSDT.jar两个文件，主要必须是做链接，不能拷贝
，否则也是白搭，你可以看一下生成出来的C文件（在~/.systemtap/cache目录下），里面
有如下内容：
: process(\"/usr/local/libexec/systemtap/libHelperSDT_amd64.so\").
人家明确写的是systemtap下的so文件，如果你拷贝到jdk目录下，直接无视，泪啊～～～，
为啥在README里面要写copy/link呢？

最后一个，必须是同一个用户的进程才行，也就是说如果你用root去分析其他用户的java进
程，不好意思，同样无视～～～

解决上面问题之后，简单的例子能跑了[fn:2]，但是我尝试了一下对weblogic做profile，
不行，一直抛如下错误，放弃～

``` shell
org.jboss.byteman.rule.exception.ExecuteException: stap_672db8a0071794f3697f800e2d70a51_22140probe_2237  : caught java.lang.ClassCircularityError: sun/reflect/MethodAccessorImpl
	at org.jboss.byteman.rule.Rule.execute(Rule.java:716)
	at org.jboss.byteman.rule.Rule.execute(Rule.java:653)
	at java.lang.ClassLoader.loadClass(ClassLoader.java)
	at sun.misc.Unsafe.defineClass(Native Method)
	at sun.reflect.ClassDefiner.defineClass(ClassDefiner.java:45)
	at sun.reflect.MethodAccessorGenerator$1.run(MethodAccessorGenerator.java:381)
	at java.security.AccessController.doPrivileged(Native Method)
	at sun.reflect.MethodAccessorGenerator.generate(MethodAccessorGenerator.java:377)
	at sun.reflect.MethodAccessorGenerator.generateMethod(MethodAccessorGenerator.java:59)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:28)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
	at java.lang.reflect.Method.invoke(Method.java:597)
	at org.jboss.byteman.rule.expression.MethodExpression.interpret(MethodExpression.java:342)
	at org.jboss.byteman.rule.Action.interpret(Action.java:144)
	at org.systemtap.byteman.helper.HelperSDT_HelperAdapter_Interpreted_1.fire(tacy-22167.btm)
	at org.systemtap.byteman.helper.HelperSDT_HelperAdapter_Interpreted_1.execute0(tacy-22167.btm)
	at org.systemtap.byteman.helper.HelperSDT_HelperAdapter_Interpreted_1.execute(tacy-22167.btm)
	at org.jboss.byteman.rule.Rule.execute(Rule.java:684)
	at org.jboss.byteman.rule.Rule.execute(Rule.java:653)
	at java.lang.ClassLoader.loadClass(ClassLoader.java)
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:249)
	at sun.rmi.server.LoaderHandler.loadClass(LoaderHandler.java:152)
	at java.rmi.server.RMIClassLoader$2.loadClass(RMIClassLoader.java:620)
	at java.rmi.server.RMIClassLoader.loadClass(RMIClassLoader.java:247)
	at sun.rmi.server.MarshalInputStream.resolveClass(MarshalInputStream.java:201)
	at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1589)
	at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1494)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1748)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1327)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:349)
	at sun.rmi.server.UnicastRef.unmarshalValue(UnicastRef.java:306)
	at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:155)
	at com.sun.jmx.remote.internal.PRef.invoke(Unknown Source)
	at javax.management.remote.rmi.RMIConnectionImpl_Stub.getAttribute(Unknown Source)
	at javax.management.remote.rmi.RMIConnector$RemoteMBeanServerConnection.getAttribute(RMIConnector.java:878)

```
# Footnotes

[fn:1] [[https://wiki.ubuntu.com/DebuggingProgramCrash][Debugging Program Crash]]

[fn:2] [[http://developerblog.redhat.com/2014/01/10/probing-java-w-systemtap/][Probing java methods with systemtap]]

[fn:3] [[http://sourceware-org.1504.n7.nabble.com/semantic-error-not-accessible-at-this-address-td245602.html][semantic error: not accessible at this address]]

[fn:4] [[http://www.swiftsoftwaregroup.com/upgrade-gcc-4-7-ubuntu-12-04/][How to upgrade GCC to 4.7+ on Ubuntu 12.04]]

[fn:5] [[https://help.ubuntu.com/community/Kernel/Compile][Ubuntu Kernel Compile]]

[fn:6] [[http://www.brendangregg.com/perf.html#Building][perf Examples]]
