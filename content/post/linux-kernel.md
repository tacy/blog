---
title: "linux kernel"
date: 2014-06-19
lastmod: 2014-06-19
draft: false
tags: ["tech", "linux", "kernel", "notes"]
categories: ["tech"]
description: "一些自己觉得关键的kernel知识，主要还是一些优化参数，记录下来，慢慢完善。。。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# linux kernel
## vm
### vfs_cache_pressure
简单说，就是控制page cache的回收，是优先回收data block还是meta block(page cache
包括这两部分），一般情况下建议还是保留meta block缓存。[fn:2]

kernel设置该值为100，也就是优先回收meta block，会导致文件浏览类的工具速度变慢，
比如find，nautil[fn:1]。

建议可以设置到50，优化性能。
``` shell
vfs_cache_pressure
------------------

This percentage value controls the tendency of the kernel to reclaim
the memory which is used for caching of directory and inode objects.

At the default value of vfs_cache_pressure=100 the kernel will attempt to
reclaim dentries and inodes at a "fair" rate with respect to pagecache and
swapcache reclaim.  Decreasing vfs_cache_pressure causes the kernel to prefer
to retain dentry and inode caches. When vfs_cache_pressure=0, the kernel will
never reclaim dentries and inodes due to memory pressure and this can easily
lead to out-of-memory conditions. Increasing vfs_cache_pressure beyond 100
causes the kernel to prefer to reclaim dentries and inodes.

Increasing vfs_cache_pressure significantly beyond 100 may have negative
performance impact. Reclaim code needs to take various locks to find freeable
directory and inode objects. With vfs_cache_pressure=1000, it will look for
ten times more freeable objects than there are.
```

### swappiness
简单说，就是先回收page cache还是先swapping anonymous page（也即是应用栈，例如
java heap），缺省设置60，也就是先swapping，这样回导致应用变得很慢。

建议可以设置到10左右，优化性能。

``` shell
swappiness

This control is used to define how aggressive the kernel will swap
memory pages.  Higher values will increase agressiveness, lower values
decrease the amount of swap.  A value of 0 instructs the kernel not to
initiate swap until the amount of free and file-backed pages is less
than the high water mark in a zone.

The default value is 60.
```

# Footnotes

[fn:1] [[http://rudd-o.com/linux-and-free-software/tales-from-responsivenessland-why-linux-feels-slow-and-how-to-fix-that][why Linux feels slow, and how to fix that]]

[fn:2] [[https://www.kernel.org/doc/Documentation/sysctl/vm.txt][linux vm documentation]]
