---
title: "perf notes"
date: 2014-06-04
lastmod: 2014-06-04
draft: false
tags: ["tech", "linux", "tools", "tuning", "trace", "performance"]
categories: ["tech"]
description: "perf使用中碰到的一些问题总结"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# perf

strace相性对于很多熟悉linux的人来说应该都有所了解，使用起来也非常方便，但是他也
有很多问题，容易导致进程响应缓慢等，当然他对于诊断简单问题，比如进程起不来，进程
挂起等，非常好用，但是更深入的性能问题，就需要用到perf了，而perf最好的教程，看大
牛Brendan Gregg写的perf相关文章[fn:1]。我这里主要在使用过程中的一些理解。

## ubuntu下的perf
我机器上的ubuntu版本是12.04，但是使用的kernel版本是

# perf

strace相性对于很多熟悉linux的人来说应该都有所了解，使用起来也非常方便，但是他也
有很多问题，容易导致进程响应缓慢等，当然他对于诊断简单问题，比如进程起不来，进程
挂起等，非常好用，但是更深入的性能问题，就需要用到perf了，而perf最好的教程，看大
牛Brendan Gregg写的perf相关文章[fn:1]。我这里主要在使用过程中的一些理解。

## ubuntu下的perf
我机器上的ubuntu版本是12.04，但是使用的kernel版本是

: linux-image-generic-lts-trusty: 已安装：  3.13.0.27.23

sudo stap -L 'process("/usr/lib/jvm/java-6-openjdk-amd64/jre/lib/amd64/server/libjvm.so").function("*")'

# Footnotes

[fn:1] [[http://www.brendangregg.com/perf.html][perf Examples]]
