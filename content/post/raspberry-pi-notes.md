---
title: "Raspberry Pi安装配置和问题记录"
date: 2013-02-10
lastmod: 2013-02-10
draft: false
tags: ["tech", "linux", "raspberrypi"]
categories: ["tech"]
description: "个人在安装配置raspberry pi过程中的记录，包括相关遇到问题的解决办法，主要包括配件购买、系统安装、基本配置。最终实现通过hdmi接口和电视连接播放视频。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

Raspberry Pi简单来说就是一个低功耗的卡片PC主机，当然他的处理能力要弱很多，可以是
用来教孩子编程，放高清（带GPU和HDMI输出），当然你可以用它来做很多事情，如果你动
手能力强，也可以自己扩展他，一切皆有可能。

Raspberry Pi主要分为两个型号，A型号是低配，B型号是高配，我用的是B板，这里都是基
于B板的描述，板子包括2个2.0的USB口，一个LAN口，HDMI/RCA/AUDIO输出口个一个，一个
MicroUSB口负责电源接入。可以看到，如果完整运行Raspberry Pi还需要一些输入输出设备
，他自己就类似一个PC主机。下面是我配置Raspberry Pi时候涉及到的一些相关步骤。

# 初步的配置目的
配置简单的Raspberry Pi，使之能通过HDMI连接TV播放视频。

# 购置的配件
Raspberry Pi板子就不用说了，淘宝到处都是，我买的是B板，其他的需要购买的配件包括：
- 一个USB键盘\\
  我买的是无线键鼠套装，方便点，毕竟未来会放在客厅用，有线总是个累赘。
- 一根HDMI线\\
  连接电视用，HDMI可以同时负责视频和音频传输，也可以用RCA接口和AUDIO接口接入电视
  ，问题也不大。只是高清自然是不行了，这样太对不起Raspberry Pi了吧。
- 一根网线\\
  当然你的有有线网络接入，如果你家里只有无线接入，可以通过购置一个带有线口的无线
  设备做repeater接入，我用的是tplink 的TL-WR710N。
- 一张SDHC卡\\
  我买的Kingston 16G class4，买之前最好去网上查查是否支持[fn:1]，据说挑卡。
- 一个放置Raspberry Pi的盒子\\
  固定和防尘用（我忘了买，挺麻烦的，不好固定，简单用个纸盒先凑合着）。

# 安装系统
第一步自然是把线接好，这个就不多说了，如果你和我一样，没有买固定的盒子，还是注意
经典和力度，比较板子还是挺脆弱的。然后下载系统镜像，Raspberry Pi官网提供几个版本
的镜像下载，我直接用的最简单的那个版本wheezy，基于Debian的Raspbian预配置版本，带
桌面和浏览器以及开发工具和一些示例的多媒体代码，安装好之后就能用，下载地址：
[[http://www.raspberrypi.org/downloads][Raspberry Pi Download]]，准备总做就做好了。

## 写入系统到SD
我用的是Ubuntu系统，机器自带SD读卡器，直接插入SD卡，检查SD卡路径：

`$sudo fdisk -l`

通常都是sdb，不清楚你就对比一下插入前后的输出，如果你的SD卡已经分区，并且自动
mount了，记得unmount它，接下来我们写入下载的系统镜像文件：

``` shell
$unzip 2012-12-16-wheezy-raspbian.zip
$sudo dd bs=1M if=2012-12-16-wheezy-raspbian of=/dev/sdb
#+end_example
```

等上几分钟或者十几分钟，视SD卡速度而定，完成之后弹出SD卡，插入Raspberry Pi卡槽中
，通过MicroUSB插入电源，注意电源一般的手机充电器都可以，前提是输出需要大于700毫
安，我试过IPad充电器也可以（2100mA），Rasberry Pi不带开关，电源插入自动开机，关
机只能拔电源，正常就可以在TV上看到Rasberry Pi的启动过程了。

## 配置Rasberry Pi
第一次系统启动自动进入Raspi-config配置界面，设置正确的timezone、
locale(en_US.UTF8)、Keyboard（Generic 105-key intel PC)、ssh enable，最后是
expand-rootfs，完成重启系统。

经过漫长的expand rootfs之后，终于来到登入终端，用pi/raspberry登入系统，用命令
startx启动进入xwindows，这时候你应该就可以正常上网了,lan通过dhcp自动获取ip地址

## 中文环境配置
缺省我用的是en_us.utf8环境，当然我也不打算修改，我需要的只是显示中文和输入中文
，只需安装字体和输入法。

``` shell
$sudo apt-get install ttf-wqy-zenhei
$sudo apt-get install scim-pinyin
```

## 自动登入xwindows[fn:2]

### 自动登入
修改/etc/inittab文件，注释行：
`1:2345:respawn:/sbin/getty --noclear 38400 tty1`

添加行:
`1:2345:respawn:/bin/login -f pi tty1 </dev/tty1 >/dev/tty1 2>&1`

### 自动启动xwindows
修改~/.bash_profile，添加如下行：

``` shell
if [ -z "$DISPLAY" ] && [ $(tty) = /dev/tty1 ]; then
  startx
fi
```

## lxde键盘快捷键
直接查看".config/openbox/lxde-rc.xml"，查看keyboard部分即可，如需添加自己的映射
，照着添加就行。

## 播放视频
系统缺省安装了omxplayer播放器，这是一个命令行播放器,GPU支持，上传一个视频，播放
视频看效果：

`$omxplayer test.avi`

另外提醒一下，omxplayer支持不启动xwindows播放视频，我之前一直以为必须启动
xwindows才行，估计其他的播放器也都一样，确实没太注意，如果你需要播放视频，你可以
直接在远程通过ssh登入Raspberry Pi，执行omxplayer命令，视频会直接显示在电视上，不
管你是不是启动了xwindows。

Enjoying!

# 远程控制
支持控制，包括终端控制的ssh和远程桌面的vnc、rdp等，我这里用了简单的共享键盘方式
，通过和客户端共享一套输入设备实现，简单说就是用我的笔记本键盘鼠标操作
Raspberry Pi的桌面（我这里是电视），实现方式：
- 修改ssh允许X11 Forward

  #+begin_example
  $sudo vi /etc/ssh/sshd_config
  #+end_example

  设置X11Forwarding的值为yes，重启sshd

- 在Raspberry Pi上安装x2x

  #+begin_example
  $sudo apt-get install x2x
  #+end_example

- 在你的客户端远程登入Raspberry Pi

  #+begin_example
  $ssh -X pi@192.168.1.104 'x2x -east -to :0'
  #+end_example

配置完成，当然这里你的Raspberry Pi上的xwindows必须启动才行，移动你的鼠标到最右边试试看，鼠标跑到电视上去
# XBMC
播放本地视频用omxplayer基本上能解决了，但是对于播放网络视频比如youku啥的，靠xwindows自然是不给力，毕竟一个Raspberry Pi的cpu太弱了，很容易跑死，XBMC自然是不二的选择。

XMBC针对Raspberry Pi有几个发布版本，集成OS，直接刷就行了，我已经刷好了系统，自然选择通过deb方式来安装了，查了查，热心人已经发布了编译好的包，参考http://michael.gorven.za.net/raspberrypi/xbmc 安装即可，对于中文支持，直接设置就可以了，先设置字体：
- 先设置字体：System - Settings - Appearance - Skin - Fonts  选择Arial based
- 然后选择中文：System - Settings - Appearance - International - Language 选择Chinese（Simple)

然后是安装视频网站插件，缺省都是国外的视频网站，要支持国内的视频网站，需要先安装插件，去这里http://code.google.com/p/xbmc-addons-chinese/ 下载插件repository.googlecode.xbmc-addons-chinese-eden.zip，然后通过XBMC安装就可以了。

播放视频的时候CPU在55%左右，还是有点偏高，还得研究研究怎么优化一下。

具体的XBMC键盘控制参考http://wiki.xbmc.org/index.php?title=Keyboard， 完整版可以参考[[https://github.com/xbmc/xbmc/blob/Frodo/system/keymaps/keyboard.xml][xbmc keyboard.xml]], 不过还是要仔细找， 我找使用键盘收藏视频找了半天，用“c”键即可。

## 字体
xbmc的缺省字体字符集应该不是gbk的，显示中文很多子都是方块，我直接用wqy替换，没找到怎么新增字体的方法：
`$sudo cp /usr/share/fonts/truetype/wqy/wqy-zenhei.ttc arial.ttf /usr/share/xbmc/media/Fonts/`

备份arial.ttf，然后重命名wqy字体即可

## 超频
Raspberry Pi运行XBMC负载很重，很容易就跑到100%，可以尝试超频，不过这个要小心，毕竟还是危险的，当然Raspberry  Pi 还是有保护措施，设置overclock时当温度超过85度时，会自动disable[fn:3]，从网上情况看，超频到900应该问题不大，下面是我的设置：

``` shell
arm_freq=900
core_freq=450
sdram_freq=450
over_voltage=2
gpu_mem=128
arm_freq_min=700
initial_turbo=30
```

设置arm_freq_min主要是因为新版本的kernel会自动调整主频，看起来好像不是很靠谱，我的长期运行在500左右，可以通过/proc/cpuinfo查看。

# 遇到的问题
- 键盘配置不对

  当你按下“|”键的时候，出来的键值不对，那应该是用了GB的键盘布局，可以通过如下方式修改：
  #+begin_example
  $sudo vi /etc/default/keyboard
  #+end_example

  修改XKBLAYOUG值为“us”，重启系统。

- 播放视频没有声音，并且视频没有满屏[fn:4]

  通过如下方式修改hdmi配置
  #+begin_example
  $sudo vi /boot/config.txt
  #+end_example
  修改hdmi_drive值为2，如果改行被注释，也需要取消注释。

  通过如下命令行播放视频：
  #+begin_example
  $omxplayer -r -o hdmi test.avi
  #+end_example

- omxplayer中文乱码

  可以直接在omxplayer中指定字体，直接使用wqy字体即可：
  #+begin_example
  $omxplayer -t 1 -p -r --font /usr/share/fonts/truetype/wqy/wqy-zenhei.ttc --align center -o hdmi test.avi
  #+end_example

- omxplayer不支持外置字幕

  目前新版本omxplayer已经支持外置字幕，可以去网站上下载[fn:5]，不过目前好像只支持utf-8，不行只能通过enca转码。

- undefined symbo问题

  “/usr/bin/omxplayer.bin: Symbol lookup error: /usr/bin/omxplayer.bin: undefined symbol: vc_xxxxxxxxxx.”

  如果你碰到类似问题，基本上是firmware没有更新，你可以通过rpi-update更新。

- 如果你遇到一些我没遇到的问题，建议参考elinux[fn:6]

# 一些有的资源
- [[http://www.raspbian.org/RaspbianMirrors][Raspberry Pi source]]
- [[http://youresuchageek.blogspot.fr/2012/09/howto-raspberry-pi-openelec-on.html][Howto Raspberry Pi: OpenELEC on Raspberry Pi]]



[fn:1] Raspberry Pi兼容SD卡查询地址：http://elinux.org/RPi_SD_cards

[fn:2] Raspberry Pi auto login : http://elinux.org/RPi_Debian_Auto_Login

[fn:3] Raspberry Pi超频设置： http://elinux.org/RPi_config.txt#Overclocking

[fn:4] [[http://www.brianhensley.net/2012/07/how-to-get-1080p-videos-running-on-my.html?spref=fb][How to get 1080p videos running on my Raspberry Pi]]

[fn:5] omxplayer download site: http://omxplayer.sconde.net/

[fn:6] elinux Raspberry Pi troubleshooting: http://elinux.org/R-Pi_Troubleshooting
