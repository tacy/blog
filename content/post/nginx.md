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
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_min_length 256;
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
