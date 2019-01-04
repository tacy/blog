---
title: "marathon in mesos"
date: 2014-08-30
lastmod: 2014-08-30
draft: false
tags: ["tech", "container", "cloud"]
categories: ["tech"]
description: "基于mesos的一个PaaS实现，试用笔记以及相关知识记录"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# mesos
记录一下最近一周的工作，最近一周一直在看PaaS，主要是结合Docker的方案，简单看了
看，成熟的东西没有，基本都有各种问题，可以参考一下lusis写的[[http://blog.lusis.org/blog/2014/06/14/paas-for-realists/][PaaS for Realists]]，

我没有去试这几个项目，未来有时间可能会做，目前我的重点在mesos上，这是[[https://www.google.com/url?sa%3Dt&rct%3Dj&q%3D&esrc%3Ds&source%3Dweb&cd%3D2&cad%3Drja&uact%3D8&ved%3D0CCsQFjAB&url%3Dhttp%253A%252F%252Fresearch.google.com%252Fpubs%252Fpub41684.html&ei%3DIvEBVM_VJI-A8QXIlIDADQ&usg%3DAFQjCNFhre6QW6LnswxdXZu-jLg3WXQ1eQ&sig2%3DFWZK9sf-fke4bqNXZI_Ikg&bvm%3Dbv.74115972,d.dGc][Omega]]的
开源实现，支持各种framework，包括hadoop/storm/spark等流行的数据处理框架，当然熟
悉这块的人也知道，资源调度器的另一个更流行的产品是YARN，但是他只支持MapReduce框
架，而mesos支持面更广，他还支持应用运行在上面，具体的比较可以参考[[http://www.quora.com/How-does-YARN-compare-to-Mesos][Quora上的帖子：
How does YARN compare to Mesos?]]，另外就是mesos支持docker实现隔离。

# marathon
marathon其实就是mesos上的一个framework，一个基于mesos的PaaS实现。通过它能部署应
用，比如java、python、ruby等。利用mesos提供的HA、FT、Scale特性，你的应用天然具备
这些特性。同时应用实现容器级别的资源隔离实现，非常方便。

# marathon + mesos + docker
部署测试了一下，还是比较理想，只是对于docker的支持还有一些问题，下面是一些需要
注意的点：

## 安装注意事项
首先mesos 0.19版本之前，支持docker都是通过External Containerization特性来支持，
需要依赖Deimos项目，0.20版本（写本文的时候，刚刚发布几天）原声支持，这里用的是
0.20版本

其次marathon目前发布的版本是0.6.1，不能支持mesos 0.20的docker特性，你需要获取源
码snapshot自己编译。

## 功能上需要注意的点
首先一个需要注意的地方是mesos对于docker网络的支持，目前之支持host模式，也就意味
着你的应用端口配置上有一些麻烦，你需要在应用启动阶段注入修改应用端口，mesos的下
一个版本（0.20.1）能支持更复杂的网络配置[fn:1]。

关于网络举例来说，缺省是bridge，docker能做端口expose，docker容器内监听的80端口，
如果外面想访问，只需把端口expose出来。这样对于运行在docker容器内的应用很方便，
不会产生端口冲突问题，但是如果是host模式，docker容器内的监听端口直接监听在宿主
机上，需要考虑同一宿主机的多实例应用端口冲突问题。

其次是LB的问题也需要注意，目前给出的示例是通过cron方式非实时更新LB配置，不是太理
想，应该可以通过callback方式来走，会比较理想。

另外一点就是[fn:2]：
``` shell
Also since we explicitly attempt to pull the image on launch,if the docker image
is only installed locally but not avaialble on the remote repository the launch
will fail as well.
```
文档里面提到，如果是本地的image是不行的，必须提交到远程仓库，否则mesos会pull失败


# Footnotes

[fn:1] [[http://tnachen.wordpress.com/2014/08/19/docker-in-mesos-0-20/][Docker on Mesos 0.20]]

[fn:2] [[http://mesos.apache.org/documentation/latest/docker-containerizer/][Docker Containerizer]]
