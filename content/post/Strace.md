---
title: "strace notes"
date: 2014-03-18
lastmod: 2014-03-18
draft: false
tags: ["tech", "linux", "strace", "tools"]
categories: ["tech"]
description: "进程trace工具，通过strace能定位一些疑难问题，比如应用挂起无响应。他不是一个程序debug工具。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Strace notes

strace是linux下的一个进程trace工具，通过strace，能够trace进程的所有系统调用，
对于解决一些棘手问题有很大帮助。典型问题比如应用挂起，响应速度慢等。

简单的使用，比如我希望看看进程进行了那些文件操作，使用方法如下：

: strace -e trace=read,write -p pid

你能看到进程所有的write和read操作，包括文件读取、网络读取、unix socket等，进程
一般挂起在IO操作上，你可以根据下面指令来获取具体的操作对象：

: lsof -p pid

通过上面两个命令，你就可以准确定位到你的问题。

一般对于strace你可能最关心的就是四个系统调用操作：open、close、read、write，这
几个操作最容易出现问题，之前就通过它定位了一个xml parser问题。

另外strace也能输出每个系统调用的时间：

: strace -T -p pid

包括子进程：

: strace -T -fp pid

调用统计：

: strace -c -fp pid

下面是一个例子，看看apache为啥起不来：

Another valuable use of strace involves looking for errors. If a program fails when it makes a system call, you'll want to be able pinpoint any errors that might have caused that failure as you troubleshoot. In all cases where a system call fails, strace will return a line with "= -1" in the output, followed by an explanation. Note: The space before -1 is very important, and you'll see why in a moment.

For this example, let's say Apache isn't starting for some reason, and the logs aren't telling ua anything about why. Let's run strace:

: strace -Ff -o output.txt -e open /etc/init.d/httpd start

Apache will attempt to restart, and when it fails, we can grep our output.txt for '= -1' to see any system calls that failed:

: $ cat output.txt | grep '= -1'

``` shell
[pid 13748] open("/etc/selinux/config", O_RDONLY|O_LARGEFILE) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/tls/i686/sse2/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/tls/i686/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/tls/sse2/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/tls/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/i686/sse2/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/i686/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/sse2/libperl.so", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/libnsl.so.1", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/usr/lib/perl5/5.8.8/i386-linux-thread-multi/CORE/libutil.so.1", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/etc/gai.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid 13748] open("/etc/httpd/logs/error_log", O_WRONLY|O_CREAT|O_APPEND|O_LARGEFILE, 0666) = -1 EACCES (Permission denied)
```

With experience, you'll come to understand which errors matter and which ones don't. Most often, the last error is the most significant. The first few lines show the program trying different libraries to see if they are available, so they don't really matter to us in our pursuit of what's going wrong with our Apache restart, so we scan down and find that the last line:

: [pid 13748] open("/etc/httpd/logs/error_log", O_WRONLY|O_CREAT|O_APPEND|O_LARGEFILE, 0666) = -1 EACCES (Permission denied)
