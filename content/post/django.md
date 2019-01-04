---
title: "django notes"
date: 2017-09-17
lastmod: 2017-09-17
draft: false
tags: ["tech", "python", "django", "notes"]
categories: ["tech"]
description: "django使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# user
python manage.py createsuperuser

# cors
[Using CORS](https://www.html5rocks.com/en/tutorials/cors/)
[django-cors-headers](https://github.com/ottoyiu/django-cors-headers)

# django debug
$ python manage.py shell
>>> import django
>>> django.setup()
from polls.models import Question, Choice
>>> q = Question(question_text="What's new?", pub_date=timezone.now())

# Save the object into the database. You have to call save() explicitly.
>>> q.save()

# migrate
1. 有时候, 你需要修改的model字段有约束, 例如外健, 而表中的数据又不能满足约束, 你可以在models定义中增加ForeignKey.db_constraint, 这样migrate的时候才不会把报错, 但是需要注意, 这样的方式foreign字段会被清空
2. 如果migrate报错, 直接去修改迁移脚本, 注意: 因为脚本执行没有事物, 错误终止之后, 重新运行会报错. 解决办法是编辑迁移脚本, 注释掉之前执行过的操作, 但是千万记得, 执行完成之后, 需要把注释去掉. 如果不去掉的话, 后面再修改model生成migrate脚本的时候, 会重新把这些被注释的操作再次生成一边, 又会报错.

# timezone
配置时区在setting.conf里面, 注意TIME_ZONE = 'Asia/Shanghai'和 USE_TZ = False, 后者觉得了存储到数据库里面的是不是UTC, 如果True, 代表UTC, 如果希望存入本地时间, 请设置为False.

# Transaction
1. django默认是autocommit
2. django默认事物隔离级别是ReadCommit
3. mysql默认隔离级别是RepeatRead
4. 可以设置django的默认隔离级别为RepeatRead
5. 可以设置django的每一个view都自动开启事物控制

``` python
DATABASES = {
    'default': {
        # 'ENGINE': 'django.db.backends.sqlite3',
        # 'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'yma',
        'USER': 'root',
        'PASSWORD': '12345678',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'ATOMIC_REQUESTS': True,
        'OPTIONS': {
            'init_command':
            'SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED'
        }
    }
}
```

# Meta.fields contains a field that isn't defined on this FilterSet: ****

1. 定义tuple的时候, 如果只有一个element, 需要写成`(element,)`
2. model确实没有这个字段

# csrf
一般, django的app默认都是在middleware里面添加了`'django.middleware.csrf.CsrfViewMiddleware'`, 也就是说, 每个view默认都会带有csrftoken这个cookie, 当你做ajax请求的时候,务必在request里面带上请求头: `X-CSRFToken`, 值就是csrftoken这个cookie的值.

参考[ django cross site request forgery protetion](https://docs.djangoproject.com/en/1.11/ref/csrf/)

axios的请求例子,参考链接[1](https://cbuelter.wordpress.com/2017/04/10/django-csrf-with-axios/), [2](https://stackoverflow.com/questions/39254562/csrf-with-django-reactredux-using-axios):

``` javascript
// axios.defaults.xsrfCookieName = 'csrftoken'
// axios.defaults.xsrfHeaderName = 'X-CSRFToken'

// 创建axios实例
const service = axios.create({
  xsrfCookieName: 'csrftoken',
  xsrfHeaderName: 'X-CSRFToken',
  baseURL: process.env.BASE_API, // api的base_url
  // baseURL: "http://127.0.0.1:8000/stock",
  timeout: 30000                  // 请求超时时间
});
```

# static file
当django运行在product模式时(setting里面DEBUG设置为False), django不会负责处理static文件, 需要nginx去找, 但是django提供了一个收集static文件的指令, 你可以通过该指令项目所有的static文件到一个指定的目录: `python manage.py collectstatic`, 目标目录定义在setting的STATIC_ROOT, 结合STATIC_URL, nginx就可以很方便的找到static文件.

# django rest framework
1. update models返回错误: `This field may not be blank `, 需要修改serializer, 允许该字段为空


# nginx + uWsgi

1. 配置virtualenv,django,新建项目,安装uWsgi
2. 写uwsgi.ini

```
# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /home/tacy_lee/lelewu
# Django's wsgi file
module          = lelewu.wsgi
# the virtualenv (full path)
home            = /home/tacy_lee/lelewu/venv

# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 4
# the socket (use the full path to be safe
socket          = /home/tacy_lee/lelewu/uwsgi.sock
# ... with appropriate permissions - may be needed
chmod-socket    = 666
# clear environment on exit
vacuum          = true
```

3. 启动uwsgi
`uwsgi --init uwsgi.ini`

4. 配置nginx

``` python
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    upstream django {
      # connect to this socket
      server unix:///home/tacy_lee/lelewu/uwsgi.sock;
    }

    server {
        listen       8000;
        server_name  104.155.216.44;
        root         /home/tacy_lee/lelewu;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
          uwsgi_pass  django;
          include     /etc/nginx/uwsgi_params;
        }

        location /static {
          alias /home/tacy_lee/lelewu/static;
        }

    }
}
```

5. 启动nginx
