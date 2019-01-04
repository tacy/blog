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
The consumer’s poll loop is designed to handle this problem. All network IO is done in the foreground when you call poll or one of the other blocking APIs. The consumer does not use any background threads. This means that heartbeats are only sent to the coordinator when you call poll. If your application stops polling (whether because the processing code has thrown an exception or a downstream system has crashed), then no heartbeats will be sent, the session timeout will expire, and the group will be rebalanced.

Poll没有后台线程发送心跳, 所以每次poll调用就是心跳发送, 这里需要注意session超时问题, 尤其是poll线程里面有业务代码的时候
