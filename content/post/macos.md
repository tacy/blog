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
# ssh over http

brew install proxychains-ng

cat /usr/local/Cellar/proxychains-ng/4.11/etc/proxychains.conf

proxychains ssh -p 1228 -ND2000 root@tacyvps-wx &

set browser to socket5 proxy 127.0.0.1:2000


# DNS
清除缓存
```
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```
