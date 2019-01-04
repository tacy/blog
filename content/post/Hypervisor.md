---
title: "Hypervisor"
date: 2012-08-31
lastmod: 2012-08-31
draft: false
tags: ["tech", "cloud", "virtualization"]
categories: ["tech"]
description: "目前主流的Hypervisor技术介绍，包括X86和各类小机，概述级别描述，具体技术细节可以参看参考章节或者搜索相关网络知识。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Oracle
## M-series（Dynamic Domains）
收购sun获得，支持dynamic domains，支持动态资源划分，可以动态分配资源(cpu/memory/io)，不支持虚拟IO设备，也不支持动态迁移。
## T-series （LDoms/Oracle VM for SPARC）
收购sun获得，支持Logic domains，一个Hypervisor层，嵌入在firmware（sun4v sparc 指令集），Logic domains 只负责资源隔离，不参与资源调度。Tseries采用CMT架构，cpu以线程为单位（一个cpu封装16core，一个core有四个硬件线程（T1）或者8个硬件线程（T2之后）），例如T1最多支持64个domain，支持虚拟IO设备和在线迁移。domain分为四种角色（Control domain/Service Domain/IO Domain/Guest Domain），Control domain完成domain管理任务。
## Oracle VM for X86
基于开源XEN实现
## Ops Center（oracle云集算产品）
server pool 目前支持：oracleVM for x86、oracleVM for sparc、oracleVM Zone，不支持Dynamic Domains
vDC 能够支持所有的server pool
Cloud Infrastructure API and Cli 只支持oracleVM for x86和oracleVM Zone

# 参考文档
1. [[http://en.wikipedia.org/wiki/Logical_Domains][Logical Domains]]
2. [[http://en.wikipedia.org/wiki/SPARC_Enterprise][Sparc enterprise]]
3. [[https://www.dropbox.com/s/b5fb6xe8b4owe2l/Introduction%20to%20dynamic%20reconfiguration%20and%20capacity%20on%20demand%20for%20sun%20sparc%20enterprise%20servers.pdf][Dynamic domain]]
