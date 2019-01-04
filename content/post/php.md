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
curl -sS https://getcomposer.org/installer | php

mv composer.phar /usr/local/bin/composer

composer config -g repo.packagist composer http://packagist.phpcomposer.com

composer self-update

composer update


apt-get update
//apt-get install php5-mcrypt
//php5enmod mcrypt
//php5:5.6-cli下安装php ext需要用docker-php-ext-install, php.ini所在的目录是/usr/local/etc/php
apt-get install -y libmcrypt-dev && docker-php-ext-install mcrypt
apt-get install -y libmemcached-dev \
    && pecl install memcached \
    && docker-php-ext-enable memcached
docker-php-ext-install pdo_mysql
//apt-get install php5-memcache
apt-get install -y zlib1g-dev && docker-php-ext-install zip


mysql -u root -ptest101 -h xxx.xxx.xxx.xxx
ERROR 1130 (HY000): Host 'xxx.xxx.xxx.xxx' is not allowed to connect to this MySQL server

Your root account, and this statement applies to any account, may only have been added with localhost access (which is recommended).

You can check this with:

SELECT host FROM mysql.user WHERE User = 'root';
If you only see results with localhost and 127.0.0.1, you cannot connect from an external source. If you see other IP addresses, but not the one you're connecting from - that's also an indication. If you see %, well then, there's another problem altogether as that is "any remote source".

You should be able to add this remote access with:

GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
FLUSH PRIVILEGES;


ERROR 1045 (28000): Access denied for user 'root'@'172.18.1.1' (using password: YES)

密码错误,重设密码, 如果你在docker里面mount了其他数据库, 需要重新设置密码, 否则会碰到上面问题, 即使密码正确


15d4a07a2357ff4df9f37fadc9656c8e/cwr1

update fq_users set password='dd4b21e9ef71e1291183a46b913ae6f2' where username='cwr1';




vendor/laravel/framework/src/Illuminate/Hashing/BcryptHasher.php check() return true

加调试日志:
Controller/LoginController.php
Log::info(Auth::user());


laravel auth::user lost auth:user丢失导致redirect to dashboard之后, 检测发现没有登入.

mg.zhijiepay.com
table:users
routes.php  -> site/login
controller/SiteController.php
SQLSTATE[42S22]: Column not found: 1054 Unknown column 'is_del' in 'where clause' (SQL: select count(*) as aggregate from `goods` where minBuyNum>=stock and `is_del` = 0)

           //elseif(md6($password.$user->salt) != $user->password){
            //    $this->message->add('error','\u5bc6\u7801\u9519\u8bef');
            //}


gys.zhijiepay.com
table:supplier
routes.php -> /
controller/LoginController.php
        //if($info->password != md5($password.$info->salt)){
            // \u767b\u5f55\u5931\u8d25\uff0c\u8df3\u56de
        //   return Redirect::back()
        //       ->withInput()
        //       ->withErrors(array('attempt' => '\u201c\u5bc6\u7801\u201d\u9519\u8bef\uff0c\u8bf7\u91cd\u65b0\u767b\u5f55\u3002'));
        //}

SQLSTATE[42S22]: Column not found: 1054 Unknown column 'is_del' in 'where clause' (SQL: select * from `supplier_category` where `sid` = 28 and `is_del` = 0)


## error
{"error":{"type":"ErrorException","message":"file_put_contents(\/meta\/services.json): failed to open stream: No such file or directory","file":"\/data\/vendor\/laravel\/framework\/src\/Illuminate\/Filesystem\/Filesystem.php","line":70}}> php artisan optimize

创建目录app/storage/meta

# mg.zhijiepay.com

public/assets里面丢东西了, 其他就是composer的时候会出一些错误, 很奇怪的app.php修改



## thinkphp
如果没有生成日志, 注意看看Runtime里面的~runtime.php, 删除它之后再试试, 这个是编译缓存, 里面有类库和配置


## laravel

class not found

http://stackoverflow.com/questions/29351295/laravel-4-2-model-eloquent-my-own-class-not-found

发现很都都是类定义本省有问题, 而laravel再加载类的时候出错并不会提示.

debug & level

http://stackoverflow.com/questions/16984519/changing-log-levels-in-laravel-4
