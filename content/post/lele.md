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
# 代理

## osx

/User/lidan/bin/qtunnel.sh 替换地址
killall -9 qtunnel

## linux

~/.config/systemd/user/qtunnel.service
systemctl --user reload-daemon
sudo systemctl --user restart qtunnel
ps -ef|grep qtunnel


## 平台

npm run build:prod
python manage.py collectstatic
python manage.py makemigrations
python manage.py migrate
