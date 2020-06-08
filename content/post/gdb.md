---
title: "gdb notes"
date: 2018-09-05
lastmod: 2018-09-05
draft: false
tags: ["tech", "debug", "gdb", "linux", "tools"]
categories: ["tech"]
description: "gdb使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
linux下最重要的debug工具，对于问题分析，非常有帮助。掌握gdb对于linux开发和故障诊断是必备的技能。

# info
(gdb) info addr <symbol>   #
(gdb) info symbol <addr>   # 显示地址的symbol名称
(gdb) info line *<addr>    # 显示地址的代码行（需要debug信息）
(gdb) info breakpoints

# breakpoint
(gdb) b *0x65ab0          # 在地址0x65ab0设置断点
(gdb) delete breakpoints 4
(gdb) condition 2 *p == 'r'

# jump
(gdb) jump *0x403EC2       # this is like set $pc = 0xADDR; continue;

# disas
获取symbol的汇编代码
(gdb) set disassembly-flavor intel
(gdb) dissa <addr>
(gdb) dissa <symbol>
(gdb) disassemble 0x400ae0,+0x10   # use +length to specify number of bytes to disassemble.

# Displays the memory contents
(gdb) x/[length][format] address_expression  # o - octal/x - hexadecimal/d - decimal/u - unsigned decimal/t - binary/f - floating point/a - address/c - char/s - string/i - instruction/b - byte/h - halfword (16-bit value)/w - word (32-bit value)/g - giant word (64-bit value)

``` shell
(gdb) x testArray
0xbfffef7b: 0x33323130
(gdb) x/c testArray
0xbfffef7b: 48 '0'
(gdb) x/5c testArray
0xbfffef7b: 48 '0' 49 '1' 50 '2' 51 '3' 52 '4'
(gdb) x/2c
0xbfffef80: 53 '5' 54 '6'
(gdb) x/wx testArray
0xbfffef7b: 0x33323130
(gdb) x/2hx testArray
0xbfffef7b: 0x3130 0x3332
(gdb) x/gx testArray
0xbfffef7b: 0x3736353433323130
(gdb) x/s testArray
0xbfffef7b: "0123456789ABCDEF"
(gdb) x/5bx testArray
0xbfffef7b: 0x30 0x31 0x32 0x33 0x34
(gdb) x/5i $pc
=> 0x8048477 <main()+58>: mov $0x0,%eax
0x804847c <main()+63>: mov 0x1c(%esp),%edx
0x8048480 <main()+67>: xor %gs:0x14,%edx
0x8048487 <main()+74>: je 0x804848e <main()+81>
0x8048489 <main()+76>: call 0x8048310 <__stack_chk_fail@plt>
```

# ptype
显示数据类型定义, 方便查看数据结构定义，查询结构成员
(gdb) whatis sock        # 打印表达式的数据类型
(gdb) ptype struct sock

# offset
(gdb) print (int)&((struct sock *) 0)->__sk_common

# strip[^1]
如果文件被strip了，objdump/readelf/nm就无法找到任何静态symbols信息，但是gdb支持查找strip出来的debuginfo文件，RHEL的所有包都包括了对应的debuginfo包，对于问题分析非常有帮助。

gdb会查找文件所在的目录和子目录，以及全局目录，全局目录定义在debug-file-directory变量中，可以方便的设置和查看：

``` shell
(gdb) show debug-file-directory
The directory where separate debug symbols are searched for is "/usr/lib/debug".
```

例如你分析/usr/bin/bash这个程序，你可以首先看看文件是否被strip：

``` shell
[tacy_lee@instance-2 ~]$ file /usr/bin/bash
/usr/bin/bash: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=9223530b1aa05d3dbea7e72738b28b1e9d82fbad, stripped
```
上面显示bash被stripped了，所有该文件里面没有任何静态symbol信息，你能安装debug包，之后，安装目录一般在/usr/lib/debug目录下，主要文件显示：

``` shell
[tacy_lee@instance-2 usr]$ rpm -ql bash-debuginfo
/usr/lib/debug
/usr/lib/debug/.build-id
/usr/lib/debug/.build-id/92
/usr/lib/debug/.build-id/92/23530b1aa05d3dbea7e72738b28b1e9d82fbad
/usr/lib/debug/.build-id/92/23530b1aa05d3dbea7e72738b28b1e9d82fbad.debug
/usr/lib/debug/usr
/usr/lib/debug/usr/bin
/usr/lib/debug/usr/bin/bash.debug
/usr/lib/debug/usr/bin/sh.debug
/usr/src/debug/bash-4.2
```
gdb可以通过buildid查找debug文件，也会在/usr/lib/debug目录下按照bash的路径查找。

build-id就是file命令显示的BuildID，目录名加文件名就是文件的buildid（'92'+'23530b1aa05d3dbea7e72738b28b1e9d82fbad')，debug目录下会按照bash的路径放置debug文件，gdb会去该目录下查找debug文件

源代码文件定义在directories变量，如果找不到源代码可以设置该变量，默认gdb好像会去搜索/usr/src/debug目录。

# coredump

## find coredump
cat /proc/sys/kernel/core_pattern
/usr/lib/systemd/systemd-coredump %p %u %g %s %t %e

Examining a core dump, Use coredumpctl to find the corresponding dump:

You need to uniquely identify the relevant dump. This is possible by specifying a PID, name of the executable, path to the executable or a journalctl predicate (see coredumpctl(1) and journalctl(1) for details). To see details of the core dumps:

Pay attention to "Signal" row, that helps to identify crash cause. For deeper analysis you can examine the backtrace using gdb:

When gdb is started, use the bt command to print the backtrace:

(gdb) bt


## set
### 指定源代码目录

``` shell
(gdb) directory ~/workspace/archlinux-linux/
Source directories searched: /home/tacy/workspace/archlinux-linux:$cdir:$cwd
```


## centos
cd /usr/lib/debug/lib/modules/`uname -r`/

[^1]: [Debugging Information in Separate Files](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)
