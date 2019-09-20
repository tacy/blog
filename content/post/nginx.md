---
title: "nginx notes"
date: 2018-07-23
lastmod: 2019-08-28
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
# keepalive
nginx的keepalive涉及到两端：一端是客户端发起的连接，另一端是nginx发起到upstream的连接，两端都存在keepalive问题

默认情况下，nginx会维持和客户端的长连，ngx_http_core_module里面有几个keepalive有关的参数：
* keepalive_requests， 设置每个keepalive连接能处理的请求数量
* keepalive_timeout，设置keepalive连接idle的时间，超出会被清除

另一端，默认情况下，nginx不会和upstream建立长连接（默认用http 1.0协议发起连接），你可以通过ngx_http_upstream_module模块和ngx_http_proxy_module模块来实现控制
* keepalive，设置每个worker process能和每个upstream server建立的长连接数量
* keepalive_requests， 设置每个keepalive连接能处理的请求数量
* keepalive_timeout，设置keepalive连接idle的时间，超出会被清除
* proxy_http_version 1.1;  设置nginx和upstream server使用的http协议版本
* proxy_set_header Connection "";  清除Connection头的内容，保证长连接

设置keepalive的主要目的就是减少连接的开销；另外就是太多短连接造成的连接处于time_out状态，导致快速消耗端口问题。当然长连接也是一体两面的，维持的这些连接也会消耗系统资源（文件句柄，内存），需要根据具体场景做灵活调整。

另外还需要提到的一点是，如果keepalive_timeout设置过长，需要考虑连接有效性问题，nginx并不会主动保持连接（发送probe机制），需要借助linux提供的tcp connection probe机制，这需要在listen声明中增加so_keepalive选项，例如`listen *:80 so_keepalive=on`

# so_linger
这个特性是控制连接断开行为，lingering_close设置为on的时候，连接采用四次挥手方式关闭连接，如果lingering_close设置为off，连接采用RESET包断开连接。

# gzip
首先, 观察请求头是否什么支持gzip: `Accept-Encoding:gzip, deflate`, 然后看response: `Content-Encoding:gzip`, 如果都有的话, 就是支持.
如果你发现部分资源支持, 部分不支持, 那就是gzip_types没有配置对该资源的压缩, 资源类型看response: `Content-Type:application/javascript`

## nginx 配置

```
    #gzip
    gzip on;
    gzip_disable "msie6";

    gzip_vary on;
    gzip_proxied expired no-cache no-store private auth;
    gzip_comp_level 2;
    gzip_buffers 16 8k;
    #gzip_http_version 1.1;
    gzip_min_length 2048;
    gzip_types text/plain text/css application/json application/x-javascript application/javascript application/octet-stream text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;
```

# 缓存 [^1]
## nginx 配置

```
        location /static {
          alias /home/tacy_lee/lelewu/static;
          expires 365d;
        }
```

## http 请求
response:

```
Cache-Control:max-age=31536000;
ETag:W/"599aa87e-d17";
Expires:Wed, 22 Aug 2018 03:19:23 GMT;
```

注意chrome F12调试的时候, Network tab里面默认是disable cache的, 所以在调试的时候容易搞混.
disable cache resquest:
```
Cache-Control:no-cache
Pragma:no-cache
```

[^1]: [http caching](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)
