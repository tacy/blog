---
title: "archlinux notes"
date: 2015-01-01
lastmod: 2019-01-04
draft: true
tags: ["tech", "linux", "archlinux", "notes"]
categories: ["tech"]
description: "archlinux使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# one-liners
*  查看某个程序消耗的内存情况
ps -eo comm,rss|grep chrome|tr -s ' '|cut -d' ' -f2|paste -sd+ - | bc

* 排序程序消耗内存情况
 ps -eo comm,rss|awk '{arr[$1]+=$2} END {for (i in arr) {print i,arr[i]}}'|sort -k 2 -g
