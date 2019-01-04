---
title: "Linux trace"
date: 2019-01-02
lastmod: 2019-01-02
draft: false
tags: ["tech", "linux", "trace", "troubleshooting"]
categories: ["tech"]

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
对于linux下的trace，我建议你参考Brendan Gregg的文章：[Choosing a linux tracer](http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html)，里面详细列出了linux下每个tracer的介绍，本文里主要介绍ftrace，讨论function相关的部分，event和计数器不涉及。

下面我们从两个维度来看看ftrace在linux中的使用，首先是根据运行态：分为kernel和userland；其次根据trace方式：分为静态分析和动态分析。

## Kernel
kernel空间下，静态分析是指tracepoint，动态分析是指kprobe
### 静态分析
是指代码里面已经埋下了探针（probe），可以对这些探针进行分析，没有探针的地方不能trace，探针列表可以通过perf查看：

``` shell
#sudo perf list tracepoint
...
syscalls:sys_enter_getcpu                          [Tracepoint event]
syscalls:sys_exit_getcpu                           [Tracepoint event]
tcp:tcp_destroy_sock                               [Tracepoint event]
tcp:tcp_probe                                      [Tracepoint event]
...
```
静态分析有两种方式：ftrace和perf

#### ftrace[^7][^8]
ftrace是linux下的主要trace框架，linux实现了一个tracefs文件系统（类似proc的虚文件系统，驻留在内存），挂载在/sys/kernel/debug/tracing目录下，里面包括所有ftrace相关的内容，ftrace就是通过配置该目录下的文件，完成tracing。

ftrace里面涉及到静态分析的除了tracepoint之外，另外还有一个部分叫function，分别介绍

##### function
traceing目录下，带function字样的文件都和function有关，文件available_filter_functions里面包括了所有能trace的function（在没有添加任何filter的情况下）。如果你需要开启function tracer（更多的tracer，查看文件available_tracers，里面包括了系统支持哪些tracer），只需要执行命令：`echo function > current_tracer`，启用之后，你能查看function trace的输出和输出格式定义：

``` shell
#cat trace|head -10
# tracer: function
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
           gdbus-1275  [001] .... 11758.901088: unix_poll <-sock_poll
```

如果你查看trace的内容，你会发现内容太多，你很难找到有价值的东西，绝大部分时候，你不是对整个系统的调用感兴趣，你只想看你关心的内容，ftrace提供了function filter来进行trace过滤：通过设置` set_ftrace_filter  set_ftrace_notrace  set_ftrace_pid`三个文件，你能过滤出来需要trace的function。

最简单的做法是看看available_filter_functions里面有些什么，里面的这些function都是可以用，例如我关心tcp_sendmsg：`echo tcp_sendmsg > set_ftrace_filter`，这样function tracer就只会记录tcp_sendmsg这个function。当然filter也支持“*”号通配符，支持三种写法：“*key/*key*/key*”。也支持模块过滤“echo :mod:ext3 > set_ftrace_filter”

另外，filter也支持“Format: <function>:<trigger>[:count]”这类的写法，通过它你能实现：某个function调用了，就触发一个预先定义的action（tracingoff/tracingon/stacktrace/snapshot/dump/cpudump），执行相关操作。例如：`echo do_trap:traceoff:3 > set_ftrace_filter`，do_trap调用三次，执行停止trace操作。

如果你需要看到调用栈，可以设置：`echo 1 > options/func_stack_trace`，但是必须注意，该操作容易导致问题，务必需要先设置filter，缩小trace范围，否则容易导致系统问题。

``` shell
[root@tacyArch tracing]# cat available_filter_functions |grep tcp_sendmsg
bpf_tcp_sendmsg
tcp_sendmsg_locked
tcp_sendmsg
[root@tacyArch tracing]# echo tcp_sendmsg > set_ftrace_filter
[root@tacyArch tracing]# echo function > current_tracer
[root@tacyArch tracing]# cat trace
# tracer: function
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
           vvv-631   [003] .... 33440.208482: tcp_sendmsg <-sock_sendmsg
  NetworkManager-555   [002] .... 33446.836585: tcp_sendmsg <-sock_sendmsg
            curl-26180 [001] .... 33449.150367: tcp_sendmsg <-sock_sendmsg
           vvv-4083  [002] .... 33456.700836: tcp_sendmsg <-sock_sendmsg

[root@tacyArch tracing]# echo > current_tracer
bash: echo: write error: Invalid argument
[root@tacyArch tracing]# echo nop >current_tracer
[root@tacyArch tracing]# echo > set_ftrace_filter
[root@tacyArch tracing]#
```

总的来说，function tracer最大的问题就是给出的信息太模糊，有用的信息太少，也无法扩展（你只能enable它）。很多时候，虽然知道function被调用，知道调用的pid和comm，但光有这些信息对于解决问题是不够的。

##### tracepoint
下面我们来看看tracepoint相关内容。tracing目录下有一个events目录，里面就是所有的kernel tracepoint，包含了所有的tracepoint，每一个tracepoint都以目录存在，目录下面是针对该tracepoint的配置文件。

trace方法就是找到对应的tracepoint，例如tcp_probe，进入到该目录下（events/tcp/tcp_probe），通过命令`echo 1 > enable`激活该探针，然后你就可以回到tracing目录下，查看trace文件内容，里面能看到trace内容，输出格式可以参考探针目录下的format文件。

另外你也能设置filter，根据条件过滤事件；你也能设置triger，当条件满足的时候，触发定义的action，例如stracktrace，输出栈到trace文件

``` shell
[root@tacyArch tcp_receive_reset]# ls
enable  filter  format  hist  id  trigger

[root@tacyArch tcp_receive_reset]# cat format
name: tcp_receive_reset
ID: 1133
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:const void * skaddr;      offset:8;       size:8; signed:0;
        field:__u16 sport;      offset:16;      size:2; signed:0;
        field:__u16 dport;      offset:18;      size:2; signed:0;
        field:__u8 saddr[4];    offset:20;      size:4; signed:0;
        field:__u8 daddr[4];    offset:24;      size:4; signed:0;
        field:__u8 saddr_v6[16];        offset:28;      size:16;        signed:0;
        field:__u8 daddr_v6[16];        offset:44;      size:16;        signed:0;
        field:__u64 sock_cookie;        offset:64;      size:8; signed:0;

print fmt: "sport=%hu dport=%hu saddr=%pI4 daddr=%pI4 saddrv6=%pI6c daddrv6=%pI6c sock_cookie=%llx", REC->sport, REC->dport, REC->saddr, REC->daddr, REC->saddr_v6, REC->daddr_v6, REC->sock_cookie

```
enable文件用来控制开关，filter可以用来做事件过滤，format是tracepoint的输出内容定义，hist做事件统计，id就是tracepoint的标识，trigger可以条件触发action。

``` shell
[root@tacyArch tracing]# echo 1 > events/tcp/tcp_receive_reset/enable
[root@tacyArch tracing]# cat trace
# tracer: nop
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            curl-26346 [001] .... 33819.317760: tcp_receive_reset: sport=44248 dport=8080 saddr=0.0.0.0 daddr=0.0.0.0 saddrv6=::1 daddrv6=::1 sock_cookie=1
            curl-26346 [001] .... 33819.317892: tcp_receive_reset: sport=49690 dport=8080 saddr=127.0.0.1 daddr=127.0.0.1 saddrv6=::ffff:127.0.0.1 daddrv6=::ffff:127.0.0.1 sock_cookie=2
[root@tacyArch tracing]# echo 0 > events/tcp/tcp_receive_reset/enable
[root@tacyArch tracing]#
```
上面的输出更友好，基本上你关心的内容都给你输出了：通过上面内容，你能知道curl从127.0.0.1:8080接收到一个reset包。

tracepoint能帮助你解决很多问题，当然tracepoint数量有限，有可能你关心的点正好没有，只能等着看新版本的kernel能否提供

#### perf
perf位于kernel源代码的tools下，linux的主要性能分析工具之一，perf提供很多功能，这里我们只谈它和tracepoint有关的部分（perf不能操作ftrace中的function tracer），以tcp_probe这个tracepoint为例，操作也很简单：

``` shell
[root@tacyArch ~]# perf record -e 'tcp:tcp_probe' -a
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.460 MB perf.data (3 samples) ]

[root@tacyArch ~]# perf script
 irq/69-brcmf_pc 19366 [003] 99975.473448: tcp:tcp_probe: src=192.168.1.21:44404 dest=203.208.xx.xx:443 mark=0 data_len=0 snd_nxt=0x66d5014 snd_una=0x66d5014 snd_cwnd=16 ssthresh=21>
 irq/69-brcmf_pc 19366 [003] 99977.000673: tcp:tcp_probe: src=192.168.1.21:59262 dest=xx.201.xx.119:80 mark=0 data_len=60 snd_nxt=0x6ec6b07d snd_una=0x6ec6b07d snd_cwnd=55 ssthresh>
 irq/69-brcmf_pc 19366 [003] 99977.057786: tcp:tcp_probe: src=192.168.1.21:59262 dest=xx.201.xx.119:80 mark=0 data_len=0 snd_nxt=0x6ec6b0a1 snd_una=0x6ec6b07d snd_cwnd=55 ssthresh=>
[root@tacyArch ~]#

```
#### 小结
静态分析方法比较简单，无需太多代码方面的知识，只需要通过配置即可完成。用户能通过对合适的tracepoint进行分析，找出有用的信息，从而解决一些疑难问题。但是静态分析受限于预先埋下的探针，另外探针输出的内容也无法增加，在一些情况下，满足不了trace的需求。

### 动态分析
下面我们来看看动态分析，动态分析，顾名思义，就是当某函数没有预先在代码中埋下探针的时候，系统能够动态给它添加探针的分析方式。在kernel中，这些动态添加的探针也被称为kprobe。kernel把所有的symbol输出在/proc/kallsyms文件中，你可以对这里面的所有symbol添加探针。kprobe可以说是linux中最牛逼的trace手段。

动态分析也支持ftrace和perf两种方式。两者的选择：优先选择perf，主要原因是perf会做错误检验，不容易导致问题，ftrace的话就需要你特别小心，这里可是在kernel层面的操作，一个不小心，系统可能会挂起或者中断。

#### ftrace[^6]
/proc/kallsyms里面所有的symbol都能添加kprobe，kprobe和function tracer无关，无需启用function tracer。你只需定义好event，写入到traceing目录下的kprobe_events文件即可，ftrace会自动在/sys/kernel/debug/tracing/events下创建kprobe目录，并在该目录下为每个kprobe创建类似tracepoint的目录，之后你只需要通过enable文件来控制启用还是禁用。kprobe event的语法定义如下：

``` shell
Synopsis of kprobe_events
-------------------------
  p[:[GRP/]EVENT] [MOD:]SYM[+offs]|MEMADDR [FETCHARGS]	: Set a probe
  r[MAXACTIVE][:[GRP/]EVENT] [MOD:]SYM[+0] [FETCHARGS]	: Set a return probe
  -:[GRP/]EVENT						: Clear a probe

 GRP		: Group name. If omitted, use "kprobes" for it.
 EVENT		: Event name. If omitted, the event name is generated
		  based on SYM+offs or MEMADDR.
 MOD		: Module name which has given SYM.
 SYM[+offs]	: Symbol+offset where the probe is inserted.
 MEMADDR	: Address where the probe is inserted.
 MAXACTIVE	: Maximum number of instances of the specified function that
		  can be probed simultaneously, or 0 for the default value
		  as defined in Documentation/kprobes.txt section 1.3.1.

 FETCHARGS	: Arguments. Each probe can have up to 128 args.
  %REG		: Fetch register REG
  @ADDR		: Fetch memory at ADDR (ADDR should be in kernel)
  @SYM[+|-offs]	: Fetch memory at SYM +|- offs (SYM should be a data symbol)
  $stackN	: Fetch Nth entry of stack (N >= 0)
  $stack	: Fetch stack address.
  $retval	: Fetch return value.(*)
  $comm		: Fetch current task comm.
  +|-offs(FETCHARG) : Fetch memory at FETCHARG +|- offs address.(**)
  NAME=FETCHARG : Set NAME as the argument name of FETCHARG.
  FETCHARG:TYPE : Set TYPE as the type of FETCHARG. Currently, basic types
		  (u8/u16/u32/u64/s8/s16/s32/s64), hexadecimal types
		  (x8/x16/x32/x64), "string" and bitfield are supported.

  (*) only for return probe.
  (**) this is useful for fetching a field of data structures.
```

我们来创建一个event：

``` shell
[root@tacyArch tracing]# echo 'p:tcp_sendmsg tcp_sendmsg' > /sys/kernel/debug/tracing/kprobe_events
[root@tacyArch tracing]# cat kprobe_events
p:kprobes/tcp_sendmsg tcp_sendmsg
[root@tacyArch tracing]# ls events/kprobes/tcp_sendmsg/
enable  filter  format  hist  id  trigger
[root@tacyArch tracing]# cat events/kprobes/tcp_sendmsg/format
name: tcp_sendmsg
ID: 1638
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:unsigned long __probe_ip; offset:8;       size:8; signed:0;

print fmt: "(%lx)", REC->__probe_ip

```

定义好之后，启用该probe进行trace（设置enable等于1）:

``` shell
[root@tacyArch tracing]# echo 1 > events/kprobes/tcp_sendmsg/enable
[root@tacyArch tracing]# cat trace |head -10
# tracer: nop
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            curl-21745 [001] ...1 27996.148112: tcp_sendmsg: (tcp_sendmsg+0x0/0x40)
[root@tacyArch tracing]# echo 0 > events/kprobes/tcp_sendmsg/enable
```
输出结果看起来和ftrace的function tracer差不多，这就尴尬了？我们想要看到调用参数

继续努力，接下来，我们看看怎么输出调用参数，先看看tcp_sendmsg代码：

（下面的操作依赖kernel debug info的安装）

``` shell
[root@tacyArch src]# gdb /lib/modules/4.19.12-arch1-1-ARCH/build/vmlinux
...
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /lib/modules/4.19.12-arch1-1-ARCH/build/vmlinux...done.
(gdb) directory /home/tacy/workspace/archlinux-linux/
Source directories searched: /home/tacy/workspace/archlinux-linux:$cdir:$cwd
(gdb) li tcp_sendmsg
1434            return err;
1435    }
1436    EXPORT_SYMBOL_GPL(tcp_sendmsg_locked);
1437
1438    int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
1439    {
1440            int ret;
1441
1442            lock_sock(sk);
1443            ret = tcp_sendmsg_locked(sk, msg, size);

```
上面的代码输出显示，tcp_sendmsg有3个调用参数。我们希望能把发送包的大小（size）输出的trace结果中。这里，你首先需要知道，在X64架构下，calling的参数存放规范（参考System V ABI AMD64[^9][^10]，找到每个register定义），根据规范，我们知道第三个参数的寄存器使用的是rdx。

下面我们重新定义一个probe，覆盖之前的：

``` shell
[root@tacyArch tracing]# echo 'probe:tcp_sendmsg tcp_sendmsg size=%dx' > kprobe_events
[root@tacyArch tracing]# cat events/kprobes/tcp_sendmsg/format
name: tcp_sendmsg
ID: 1639
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:unsigned long __probe_ip; offset:8;       size:8; signed:0;
        field:u64 size; offset:16;      size:8; signed:0;

print fmt: "(%lx) size=0x%Lx", REC->__probe_ip, REC->size
[root@tacyArch tracing]# echo 1 > events/kprobes/tcp_sendmsg/enable
[root@tacyArch tracing]# cat trace
# tracer: nop
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            curl-22314 [000] ...1 29227.292645: tcp_sendmsg: (tcp_sendmsg+0x0/0x40) size=0x4c
[root@tacyArch tracing]# echo 0 > events/kprobes/tcp_sendmsg/enable
```
干的不错，能看到包大小了，能不能继续进一步，看看socket信息呢？

``` shell
(gdb) print (int)&((struct sock *)0)->__sk_common
$1 = 0
(gdb) print (int)&((struct sock_common *)0)->skc_daddr
$2 = 0
(gdb) print (int)&((struct sock_common *)0)->skc_rcv_saddr
$3 = 4
(gdb) print (int)&((struct sock_common *)0)->skc_dport
$4 = 12
(gdb) print (int)&((struct sock_common *)0)->skc_num
$5 = 14
[root@tacyArch tracing]# echo 'probe:tcp_sendmsg tcp_sendmsg dst=+0(%di):x32 dport=+12(%di):x16 src=+4(%di):x32 sport=+14(%di):x16 size=%dx' > kprobe_events
[root@tacyArch tracing]# echo 1 > events/kprobes/tcp_sendmsg/enable
[root@tacyArch tracing]# tail -2 trace
            curl-25876 [001] ...1 37339.320598: tcp_sendmsg: (tcp_sendmsg+0x0/0x40) dst=0x64e959ca dport=0x5000 src=0x670aa8c0 sport=0xa4f0 size=0x4c
            curl-25878 [000] ...1 37340.118164: tcp_sendmsg: (tcp_sendmsg+0x0/0x40) dst=0x64e959ca dport=0x5000 src=0x670aa8c0 sport=0xa4f2 size=0x4c
[root@tacyArch tracing]# echo 0 > events/kprobes/tcp_sendmsg/enable
```
这个就厉害啦，少年，有没有跃跃欲试的感觉，还不赶紧操练一下去（写个脚本把地址转换一下呗）！

总结一下就是，终于寻得屠龙宝刀在手。。。唯一的遗憾就是不能做复杂程序处理。

#### perf
首先需要说明的是，perf无法操作ftrace里面的function tracer，它只能定义和操作kprobe（当然也可以定义uprobe，后面会有描述），perf-probe其实操作的也是/sys/kernel/debug/tracing/kprobe_event。但是通过perf-probe定义probe的好处就是会有校验，不容易出错

还是以tcp_sendmsg为例，添加和使用probe的方式：

``` shell
[root@tacyArch ~]#perf probe --add tcp_sendmsg  // 无参数版本
Added new event:
  probe:tcp_sendmsg    (on tcp_sendmsg)

You can now use it in all perf tools, such as:

        perf record -e probe:tcp_sendmsg -aR sleep 1


[root@tacyArch ~]# perf probe --list
  probe:tcp_sendmsg    (on tcp_sendmsg@net/ipv4/tcp.c)

[root@tacyArch ~]# perf probe -d probe:tcp_sendmsg  //删除

[root@tacyArch tracing]# perf probe 'tcp_sendmsg tcp_sendmsg dst=+0(%di):x32 dport=+12(%di):x16 src=+4(%di):x32 sport=+14(%di):x16 size=%dx'  //有参数

```
有了probe之后，你就可以对该probe进行trace操作了，操作方法和静态分析中的perf类似

如果你的kernel安装了debug info，perf能更简单的定义probe，同样上面显示socket信息的tcp_sendmsg，如果有debuginfo，你不用去找寄存器和偏移，直接使用变量名：

``` shell
[root@tacyArch tracing]# perf probe 'tcp_sendmsg size sk->__sk_common.skc_daddr sk->__sk_common.skc_dport sk->__sk_common.skc_rcv_saddr sk->__sk_common.skc_num'

```
另外，在有debuginfo的情况下，你可以不借助gdb，直接用perf查看代码和变量：
``` shell
# perf probe -L tcp_sendmsg:0+10   # 查看tcp_sendmsg代码0~10行，如果你的源文件放在自己目录里面，通过-x参数指定
<tcp_sendmsg@/home/tacy/workspace/archlinux-linux/net/ipv4/tcp.c:0>
      0  int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
      1  {
      2         int ret;

      4         lock_sock(sk);
      5         ret = tcp_sendmsg_locked(sk, msg, size);
      6         release_sock(sk);

      8         return ret;
      9  }
         EXPORT_SYMBOL(tcp_sendmsg);

         /*

# perf probe -V tcp_sendmsg        # 查看tcp_sendmsg参数
Available variables at tcp_sendmsg
        @<tcp_sendmsg+0>
                int     ret
                size_t  size
                struct msghdr*  msg
                struct sock*    sk

# perf probe --add 'tcp_sendmsg size'  # 使用tcp_sendmsg参数，输出到结果中
# perf probe -V tcp_sendmsg:+9  # 可以查看函数内部变量，内部变量也能输出
```
#### 小结
动态分析方法和静态分析方法相比，需要你具备代码级别的知识，对kernel代码越熟悉，对linux的tracer就像玩一样，感觉没啥解决不了的问题，这方面可以参考大牛Brendan Gregg。

同时，通过kprobe，也是让你学习了解kernel的一个最佳途径，无需修改代码，直接进行debug。

## Userland
Usersland的trace同样包括静态和动态两部分，但是这两部分我们也统一称为uprobe[^5]，你可以在ftrace目录（/sys/kernel/debug/tracing）下的uprobe_events文件中找到你定义的所有uprobe event。

uprobe event定义：

``` shell
Synopsis of uprobe_tracer
-------------------------
  p[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : Set a uprobe
  r[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : Set a return uprobe (uretprobe)
  -:[GRP/]EVENT                           : Clear uprobe or uretprobe event

  GRP           : Group name. If omitted, "uprobes" is the default value.
  EVENT         : Event name. If omitted, the event name is generated based
                  on PATH+OFFSET.
  PATH          : Path to an executable or a library.
  OFFSET        : Offset where the probe is inserted.

  FETCHARGS     : Arguments. Each probe can have up to 128 args.
   %REG         : Fetch register REG
   @ADDR	: Fetch memory at ADDR (ADDR should be in userspace)
   @+OFFSET	: Fetch memory at OFFSET (OFFSET from same file as PATH)
   $stackN	: Fetch Nth entry of stack (N >= 0)
   $stack	: Fetch stack address.
   $retval	: Fetch return value.(*)
   $comm	: Fetch current task comm.
   +|-offs(FETCHARG) : Fetch memory at FETCHARG +|- offs address.(**)
   NAME=FETCHARG     : Set NAME as the argument name of FETCHARG.
   FETCHARG:TYPE     : Set TYPE as the type of FETCHARG. Currently, basic types
		       (u8/u16/u32/u64/s8/s16/s32/s64), hexadecimal types
		       (x8/x16/x32/x64), "string" and bitfield are supported.
```

### 静态分析
一般称为USDT Probe，需要预先在代码中加探针（开发人员可以参考[Adding User Space Probing to an Application](https://www.sourceware.org/systemtap/wiki/AddingUserSpaceProbingToApps)，学习如何在代码里面埋探针），这类探针技术也叫DTRACE，来源于Solaris，很多软件支持，但是需要在编译时启用，例如java，就需要在编译时启用enable-dtrace编译参数(启用dtrace编译需要依赖systemtap-sdt-dev包，至少jdk编译需要）

同样，我们也可以通过ftrace和perf两种方式操作它。但是ftrace操作方法复杂（主要是需要小心操作地址），方法参考[Hacking Linux USDT with Ftrace](http://www.brendangregg.com/blog/2015-07-03/hacking-linux-usdt-ftrace.html)。

下面我们用openjdk为例，说明用户空间的静态分析方法。

首先需要你的JDK启用了enable-dtrace，其次启动java应用的时候加上参数`-XX:+ExtendedDTraceProbes`，查看USDT probe：

``` shell
readelf -n /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so
Displaying notes found in: .note.gnu.build-id
  Owner                 Data size       Description
  GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: deaa14871e0101bc717a00461519385ebdb010fd

Displaying notes found in: .note.stapsdt
  Owner                 Data size       Description
  stapsdt              0x0000004d       NT_STAPSDT (SystemTap probe descriptors)
    Provider: hotspot
    Name: class__loaded
    Location: 0x00000000005b5d7a, Base: 0x0000000000dbb1b2, Semaphore: 0x0000000000000000
    Arguments: 8@%r13 -4@%r14d 8@%rax 1@%r12b
  stapsdt              0x0000004c       NT_STAPSDT (SystemTap probe descriptors)
    Provider: hotspot
    Name: class__unloaded
    Location: 0x00000000005b5f11, Base: 0x0000000000dbb1b2, Semaphore: 0x0000000000000000
    Arguments: 8@%rbx -4@%r13d 8@%rax 1@$0
  stapsdt              0x00000071       NT_STAPSDT (SystemTap probe descriptors)
    Provider: hotspot
    Name: method__compile__begin

```
确保出处中有stapsdt字样，说明你的jdk启用了dtrace。

#### ftrace
我们用method__entry这个probe为例，讲述如何通过ftrace来trace该probe。参考前面uprobe event的定义`p[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS] : Set a uprobe`，我们需要找到method__entry在libjvm.so中的位置，可以通过readelf找到：

``` shell
[root@tacyArch tacy]# readelf -n /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so |tail +2659 -|head -5 -
  stapsdt              0x00000062       NT_STAPSDT (SystemTap probe descriptors)
    Provider: hotspot
    Name: method__entry
    Location: 0x0000000000bd21e5, Base: 0x0000000000dbb1b2, Semaphore: 0x0000000000000000
    Arguments: -8@%rax 8@%rdx -4@%ecx 8@%rsi -4@%edi 8@%r8 -4@%r9d
```
method__entry的位置在0x0000000000bd21e5，因为libjvm.so是一个共享库，所以这个地址可以直接使用（如果非共享库，你还需要找到程序的base加载地址，通过公式：location - base load address，计算得到偏移）。

readelf的输出中，可以看到method__entry的arguments，总共有7个（这里我也没搞明白为啥register的使用不用遵守System V ABI AMD64规范），具体每个参数的定义就需要去看代码了，我机器上找不到stp文件，只能看在线[stp](https://github.com/mpujari/systemtap-tapset-openjdk9/blob/master/tapset-1.8.0/hotspot-1.8.0.stp.in#L411)，引用在下面：

``` shell
/* hotspot.method_entry (extended probe)
   Triggers when a method is entered.
   Sets thread_id to the current java thread id, class to the name of
   the class, method to the name of the method, and sig to the
   signature string of the method.
   Needs -XX:+ExtendedDTraceProbes.
*/
probe hotspot.method_entry =
  process("@ABS_CLIENT_LIBJVM_SO@").mark("method__entry"),
  process("@ABS_SERVER_LIBJVM_SO@").mark("method__entry")
{
  name = "method_entry";
  thread_id = $arg1;
  class = user_string_n($arg2, $arg3);
  method = user_string_n($arg4, $arg5);
  sig = user_string_n($arg6, $arg7);
  probestr = sprintf("%s(thread_id=%d,class='%s',method='%s',sig='%s')",
		     name, thread_id, class, method, sig);
}
```
大致搞清楚每个参数的定义：arg1是线程id，arge2看起来里面包含了classname，arg3是classname的长度，依次类推，下面我们就可以来定义probe了：

``` shell
[root@tacyArch tacy]# echo 'p:uprobes/method_entry /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so:0x0000000000bd21e5 arg1=%ax:s64 arg2=+0(%dx):string arg3=%cx:s32 arg4=+0(%si):string arg5=%di:s32 arg6=+0(%r8):string arg7=%r9:s32' > uprobe_events
```

搞定，可以测试一下效果：

``` shell
[root@tacyArch tracing]# echo 1 >events/uprobes/method_entry/enable
[root@tacyArch tracing]# echo 0 >events/uprobes/method_entry/enable
[root@tacyArch tracing]# cat trace|grep hello
 http-nio-8080-e-31172 [003] d... 51704.118354: method_entry: (0x7f80e000d1e5) arg1=24 arg2="hello/Greeting<init>(JLjava/lang/String;)V" arg3=14 arg4="<init>(JLjava/lang/String;)V" arg5=6 arg6="(JLjava/lang/String;)V" arg7=22
 http-nio-8080-e-31172 [003] d... 51704.134194: method_entry: (0x7f80e000d1e5) arg1=24 arg2="hello/GreetinggetId()J" arg3=14 arg4="getId()J" arg5=5 arg6="()J" arg7=3
 http-nio-8080-e-31172 [003] d... 51704.134229: method_entry: (0x7f80e000d1e5) arg1=24 arg2="hello/GreetinggetContent&()Ljava/lang/String;" arg3=14 arg4="getContent&()Ljava/lang/String;" arg5=10 arg6="()Ljava/lang/String;" arg7=20
 http-nio-8080-e-31173 [002] d... 51704.628421: method_entry: (0x7f80e000d1e5) arg1=25 arg2="hello/GreetinggetId()J" arg3=14 arg4="getId()J" arg5=5 arg6="()J" arg7=3
 http-nio-8080-e-31173 [002] d... 51704.628476: method_entry: (0x7f80e000d1e5) arg1=25 arg2="hello/GreetinggetContent&()Ljava/lang/String;" arg3=14 arg4="getContent&()Ljava/lang/String;" arg5=10 arg6="()Ljava/lang/String;" arg7=20
```

上面我们用grep来过滤我们想看到的class，其实不用这么麻烦，参考events的filter，你可以直接过滤掉你不想看到的内容：

``` shell
[root@tacyArch tracing]# echo 'arg2 ~ "hello*"' > events/uprobes/method_entry/filter
```

牛，java的方法调用一览无遗，用这个来profile method响应时间，不用对应用做任何调整，想想都觉得不可思议（如果你需要获取每个method的调用时间，你需要定义一个return probe，参考uprobe event定义）

#### perf
方法参考[Static User Tracing](http://www.brendangregg.com/perf.html#StaticUserTracing)，我简单描述一下jdk实现

为了perf能操作USDT Probe，需要把它添加到perf buildcache：

``` shell
/****************** sdt需要先添加到perf里面：**************/
/* 注意，perf操作sdt，建议切换到root运行，否则可能出一些奇怪问题，例如无法添加sdt到probe */
[root@tacyArch tracing]# perf buildid-cache --add /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so

[root@tacyArch tracing]# perf list sdt

List of pre-defined events (to be used in -e):

  sdt_hotspot:class__initialization__clinit          [SDT event]
  sdt_hotspot:class__initialization__concurrent      [SDT event]
  sdt_hotspot:class__initialization__end             [SDT event]
  sdt_hotspot:class__initialization__erroneous       [SDT event]
  sdt_hotspot:class__initialization__error           [SDT event]
  sdt_hotspot:class__initialization__recursive       [SDT event]
  sdt_hotspot:class__initialization__required        [SDT event]
  sdt_hotspot:class__initialization__super__failed   [SDT event]
  sdt_hotspot:class__loaded                          [SDT event]
  sdt_hotspot:class__unloaded                        [SDT event]
  sdt_hotspot:compiled__method__load                 [SDT event]
  sdt_hotspot:compiled__method__unload               [SDT event]
  sdt_hotspot:gc__begin                              [SDT event]
  sdt_hotspot:gc__end                                [SDT event]
  sdt_hotspot:mem__pool__gc__begin                   [SDT event]
  sdt_hotspot:mem__pool__gc__end                     [SDT event]
  sdt_hotspot:method__compile__begin                 [SDT event]
  sdt_hotspot:method__compile__end                   [SDT event]
```

添加到buildcache之后，你才能用他们定义uprobe，执行trace操作，下面用gc__begin示例（method__entry获取参数方法和ftrace一致，只是不需要去找offset）：

``` shell
[root@tacyArch tacy]# perf probe --add sdt_hotspot:gc__begin                                                                                                                 [20/1846]
Added new events:
  sdt_hotspot:gc__begin (on %gc__begin in /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so)
  sdt_hotspot:gc__begin_1 (on %gc__begin in /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so)
  sdt_hotspot:gc__begin_2 (on %gc__begin in /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so)
  sdt_hotspot:gc__begin_3 (on %gc__begin in /usr/lib/jvm/java-10-openjdk/lib/server/libjvm.so)

You can now use it in all perf tools, such as:

        perf record -e sdt_hotspot:gc__begin_3 -aR sleep 1


[root@tacyArch tacy]# perf record -e 'sdt_hotspot:gc__begin,sdt_hotspot:gc__begin_1,sdt_hotspot:gc__begin_2,sdt_hotspot:gc__begin_3' -a

^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.706 MB perf.data (11 samples) ]

[root@tacyArch tacy]# perf script
       VM Thread  7050 [000] 97761.385404: sdt_hotspot:gc__begin_2: (7f2cebede940) arg1=0
       VM Thread  6178 [000] 97762.596125: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [003] 97762.863885: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [002] 97763.374132: sdt_hotspot:gc__begin_3: (7f5430da5e31)
       VM Thread  6178 [002] 97763.374145: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [002] 97763.388108: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [001] 97764.320487: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [000] 97765.442266: sdt_hotspot:gc__begin_3: (7f5430da5e31)
       VM Thread  6178 [000] 97765.442274: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [001] 97765.462569: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0
       VM Thread  6178 [002] 97766.663314: sdt_hotspot:gc__begin_2: (7f5430da5940) arg1=0

```

### 动态分析
动态添加uprobe和操作USDT probe差不多，简单说一下perf的方法

``` shell
[root@tacyArch tracing]# perf probe -x /lib64/libc.so.6 -F|grep malloc|head -10
cache_malloced
malloc
malloc_check
malloc_consolidate
malloc_get_state@GLIBC_2.2.5
malloc_hook_ini
malloc_info
malloc_init_state
malloc_printerr
malloc_set_state@GLIBC_2.2.5

[root@tacyArch tracing]# perf probe -x /lib64/libc.so.6 malloc
Added new event:
  probe_libc:malloc    (on malloc in /usr/lib/libc-2.28.so)

You can now use it in all perf tools, such as:

        perf record -e probe_libc:malloc -aR sleep 1

[root@tacyArch tracing]# cd
[root@tacyArch ~]# perf record -e probe_libc:malloc -a
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.900 MB perf.data (3349 samples) ]

[root@tacyArch ~]# perf script |head -10
     soffice.bin   312 [001] 53221.883430: probe_libc:malloc: (7f2503f9a8d0)
     soffice.bin   312 [001] 53221.883453: probe_libc:malloc: (7f2503f9a8d0)
            java 30971 [003] 53221.941838: probe_libc:malloc: (7f3682e238d0)
            java 30971 [003] 53221.941875: probe_libc:malloc: (7f3682e238d0)
            java 30971 [003] 53221.941887: probe_libc:malloc: (7f3682e238d0)
            java 30971 [003] 53221.966116: probe_libc:malloc: (7f3682e238d0)
            java 30971 [003] 53221.966164: probe_libc:malloc: (7f3682e238d0)
            java 30971 [003] 53221.966260: probe_libc:malloc: (7f3682e238d0)
            java 30971 [003] 53221.966272: probe_libc:malloc: (7f3682e238d0)
     soffice.bin   312 [001] 53221.983791: probe_libc:malloc: (7f2503f9a8d0)
```
## 总结
上面介绍的内容，操作起来有一定复杂度，大牛Brendan Gregg提供了[perf-tools](https://github.com/brendangregg/perf-tools)工具集来简化操作，里面提供了很多非常实用的工具，大力推荐。另外大牛的[blog](http://www.brendangregg.com/blog/2015-07-03/hacking-linux-usdt-ftrace.html)建议反复阅读

ftrace最大的问题是不可编程，无法做更多扩展，但是它不需要太新的kernel，对目前很多生产环境来说，这非常关键，而且ftrace也在持续改进，建议follow大牛@srostedt（kernel ftrace maintainer）。

至于目前最热门的ebpf/bcc[^12]，强烈建议学习，优点就是一句话：可编程。kernel至少都需要4.1（推荐4.8以上），RHEL7.6官方已经支持[^11]。

## 相关知识
### assambly & gdb
### dynamic symbol / static symbol
dynamic symbol是指动态链接的程序对外依赖的共享库symbol，程序运行时必须。static symbol这个需要编译时打开`-g`开关才能看到，调试用


[^5]: [uprobe tracer](https://www.kernel.org/doc/Documentation/trace/uprobetracer.txt)

[^6]: [kprobe trace](https://www.kernel.org/doc/Documentation/trace/kprobetrace.txt)

[^7]: [ftrace document](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)

[^8]: [Secrets of the Ftrace function tracer](https://lwn.net/Articles/370423/)

[^9]: [NASM Tutorial](http://cs.lmu.edu/~ray/notes/nasmtutorial/)

[^10]: [X86 psABI](https://github.com/hjl-tools/x86-psABI/wiki/x86-64-psABI-1.0.pdf)

[^11]: [bcc install](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

[^12]: [iovisor bcc](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

[^13]: [Choosing a Linux Tracer (2015)](http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html)
