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
