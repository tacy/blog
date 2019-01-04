---
title: "Openstack quastion diagnose"
date: 2012-07-31
lastmod: 2012-07-31
draft: false
tags: ["tech", "cloud", "openstack"]
categories: ["tech"]
description: "使用openstack过程中碰到的一下问题，做一个简单的记录，主要针对问题，不涉及到概念介绍，如果你碰巧碰到同样的问题，希望能让你少走弯路。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---
# 在单nova-network service下错误配置multi-host导致的问题

在配置openstack的时候，由于开始的时候很多东西没摸清楚，导致在配置second compute node的时候出现问题，表现为在该节点创建vm的时候无法创建成功，总是停留在Networking这个地方，等待一段时候之后抛如下异常：
#+BEGIN_SRC sh
2012-07-23 13:10:31 ERROR nova.rpc.common [req-f354d5ad-17c5-409e-adfd-ec1b89dec11e 568399d8b5cd4bf29864c4540bebd983 c6615626265b4ece99980509501541f6] Timed out waiting for RPC response: timed out
2012-07-23 13:10:31 TRACE nova.rpc.common Traceback (most recent call last):
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/nova/rpc/impl_kombu.py", line 490, in ensure
2012-07-23 13:10:31 TRACE nova.rpc.common     return method(*args, **kwargs)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/nova/rpc/impl_kombu.py", line 567, in _consume
2012-07-23 13:10:31 TRACE nova.rpc.common     return self.connection.drain_events(timeout=timeout)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/kombu/connection.py", line 175, in drain_events
2012-07-23 13:10:31 TRACE nova.rpc.common     return self.transport.drain_events(self.connection, **kwargs)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/kombu/transport/pyamqplib.py", line 238, in drain_events
2012-07-23 13:10:31 TRACE nova.rpc.common     return connection.drain_events(**kwargs)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/kombu/transport/pyamqplib.py", line 57, in drain_events
2012-07-23 13:10:31 TRACE nova.rpc.common     return self.wait_multi(self.channels.values(), timeout=timeout)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/kombu/transport/pyamqplib.py", line 63, in wait_multi
2012-07-23 13:10:31 TRACE nova.rpc.common     chanmap.keys(), allowed_methods, timeout=timeout)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/kombu/transport/pyamqplib.py", line 120, in _wait_multiple
2012-07-23 13:10:31 TRACE nova.rpc.common     channel, method_sig, args, content = read_timeout(timeout)
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/kombu/transport/pyamqplib.py", line 94, in read_timeout
2012-07-23 13:10:31 TRACE nova.rpc.common     return self.method_reader.read_method()
2012-07-23 13:10:31 TRACE nova.rpc.common   File "/usr/lib/python2.7/dist-packages/amqplib/client_0_8/method_framing.py", line 221, in read_method
2012-07-23 13:10:31 TRACE nova.rpc.common     raise m
2012-07-23 13:10:31 TRACE nova.rpc.common timeout: timed out
#+END_SRC

跟了半天，发现在做allocate network的时候，rpc.call调用publish到的topic竟然是network.node2，这样自然没人处理这个message，找了半天，没发现nova.conf里面配置了multi-host，在网上翻了翻，看到一些类似的错误，但是没发现明确的解释，最好跑到mysql里面去看了看networks表，汗，里面的multi-host竟然设置为1。手动update一下，问题解决了

想想，这应该是刚开始按照文档配置的时候，没有理解multi-host，手贱在创建网络的时候顺手带上了multi-host=T，希望给遇到同样问题的同学一点有用的信息。
