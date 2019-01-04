---
title: "XBMC插件之网易公开课"
date: 2013-07-23
lastmod: 2013-07-23
draft: true
tags: ["tech", "python", "xmbc", "raspberrypi"]
categories: ["tech"]
description: "自己为XBMC写的一个网易公开课插件，满足课程分类/学校分类/查找功能"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# Performance
## bcc
这个需要kernel4.8/4.9以上, rhel7.6 包括了bcc工具集合, 目前还没有release(2018-11-28)
安装aur的bcc, 工具目录位于/usr/share/bcc/

## perf-tools
这个要求不高, 3.2就支持了, 在很多生产服务器上可以很方便使用
直接'git clone --depth 1 https://github.com/brendangregg/perf-tools', 我把它放在~/workspace/tools/perf-tools里面

libjvm.so需要安装jvm的debug info

## java
### btrace
### flamegraph
### perf
