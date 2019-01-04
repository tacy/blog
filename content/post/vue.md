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
# 问题
1. 如何设置根据table cell值设置颜色


# 遇到的问题
Q: 选择框选择了不回显.
A: 由于v-model的变量没有定义, 所以无法显示.

# 调试
Q: 如何页面数据
A: 选择页面的root element, 就可以看到, 当然是要进到vue debugtool


# nodejs
## Install Node.js 7.x repository
curl -sL https://rpm.nodesource.com/setup_8.x | bash -

## Install Node.js and npm
yum install nodejs

npm install

npm run build:prod
