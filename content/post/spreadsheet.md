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
# tips
## query

不显示title: "select * offset 1 label A '', B ''"

# vlookup
注意格式问题, 公式计算出来的列当成search key的时候, 需要从数据源修正格式才行, value()可以转换text到number

另外vlookup的最后一个参数false不要省略.
