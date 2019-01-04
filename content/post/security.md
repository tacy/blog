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
# SSL & TLS & HTTPS

[参考链接](http://security.stackexchange.com/questions/5126/whats-the-difference-between-ssl-tls-and-https)

TLS is the new name for SSL. Namely, SSL protocol got to version 3.0; TLS 1.0 is "SSL 3.1". TLS versions currently defined include TLS 1.1 and 1.2. Each new version adds a few features and modifies some internal details. We sometimes say "SSL/TLS".

HTTPS is HTTP-within-SSL/TLS. SSL (TLS) establishes a secured, bidirectional tunnel for arbitrary binary data between two hosts. HTTP is a protocol for sending requests and receiving answers, each request and answer consisting of detailed headers and (possibly) some content. HTTP is meant to run over a bidirectional tunnel for arbitrary binary data; when that tunnel is an SSL/TLS connection, then the whole is called "HTTPS".

To explain the acronyms:

"SSL" means "Secure Sockets Layer". This was coined by the inventors of the first versions of the protocol, Netscape (the company was later bought by AOL).
"TLS" means "Transport Layer Security". The name was changed to avoid any legal issues with Netscape so that the protocol could be "open and free" (and published as a RFC). It also hints at the idea that the protocol works over any bidirectional stream of bytes, not just Internet-based sockets.
"HTTPS" is supposed to mean "HyperText Transfer Protocol Secure", which is grammatically unsound. Nobody, except the terminally bored pedantic, ever uses the translation; "HTTPS" is better thought of as "HTTP with an S that means SSL". Other protocol acronyms have been built the same way, e.g. SMTPS, IMAPS, FTPS... all of them being a bare protocol that "got secured" by running it within some SSL/TLS.
