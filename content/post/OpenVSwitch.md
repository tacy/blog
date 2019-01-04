---
title: "OpenVSwitch notes"
date: 2012-08-31
lastmod: 2012-08-31
draft: false
tags: ["tech", "network", "virtualization"]
categories: ["tech"]
description: "openvswitch简单笔记，包括介绍和一些技术细节，使用过程中的一些操作记录和理解，更多的是方便自己以后能做一个概括性的了解。不定期完善，类似wikipedia。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# tunnel
## GRE (Generic Routing Encapsulation)
采用GRE隧道协议实现网络虚拟化，基于IP网络，由于NIC的性能优化特性都工作在tcp层，无法利用，导致性能地下。
## STT (stateless Transport Tunnelling Protocol
解决网络虚拟化提出的隧道协议解决方案，由Nicira提出（参考IEFT：[[http://tools.ietf.org/html/draft-davie-stt-01][STT draft）]]。相比GRE的优势，主要在于STT能利用NIC的特性（例如TSO，LRO），提升传输效率。对比GRE性能提升4倍。


# 参考文档
1. [[http://networkheresy.com/2012/06/08/the-overhead-of-software-tunneling][the overhead of software tunneling]]
