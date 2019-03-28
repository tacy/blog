---
title: "archlinux notes"
date: 2015-01-01
lastmod: 2019-01-04
draft: false
tags: ["tech", "linux", "archlinux", "notes"]
categories: ["tech"]
description: "archlinux使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# my archlinux machine
## baseline
archlinux包更新非常, 如果你经常需要安装包, 会频繁出现新安装的包更新了依赖包, 但是原来依赖这个依赖包的组件可能就不能运行了, 这种情况下, 导致你需要频繁的升级整个系统.

我的做法是把仓库固定在某个时间点上, 例如'2018-11-29', 让pacman的server只用这个路径的包, 这样就可以解决这个问题, 编辑/etc/pacman.d/mirrorlist, 只用下面这个server, 其他全部注释掉

``` shell
Server=https://archive.archlinux.org/repos/2018/11/29/$repo/os/$arch
```

## gnome
### gnome-shell
aur: chrome-gnome-shell-git
chrome plugin: chrome-gnome-shell
extension: pixel-saver & hidetopbar & User themes & no-title-bar
themes aur: gtk-theme-arc-git
选择themes用tweak tools

### gnome-terminal
* .config/gtk3.0/gtk.css

``` shell
terminal-window notebook > header.top button {
    padding: 0 0 0 0;
    background-image: none;
    border: 0;
}
terminal-window notebook > header.top > tabs > tab {
    margin: 0 0 0 0;
    padding: 0 0 0 0;
}
terminal-window notebook > header.top > tabs > tab label {
    padding: 0 0 0 0;
    margin: 0 0 0 0;
}
```
* profile
theme variant -> light
text and background color -> white on black
Palette -> linux console
limit scrollback to   -> 10000
shortcuts -> switch to tab1 -> Ctrl_1



# kernel #
启动加载模块在/etc/modules-load.d/modules.conf里面添加
# systemd #
显示service文件内容指令: `systemctl cat servicename`
显示当前运行的service: `systemctl -t service --state running`

# journal
Journal size limit, edit `/etc/systemd/journald.conf`, add line: `SystemMaxUse=50M`

clean journal files manually: `journalctl --vacuum-size=100M` or `journalctl --vacuum-time=2weeks`

journal file no presistent, edit `/etc/systemd/journald.conf`:
```
Storage=volatile
RuntimeMaxUse=25
```
# env

## MAN PATH
export MANPATH=$MANPATH:/your/path

## graphical applications evn variables
Applications running on Wayland may to use systemd user environments variables instead, as Wayland does not initiate any Xorg related files:
```
~/.config/environment.d/envvars.conf
PATH=$PATH:~/scripts
GUIVAR=value
```


# Hardware INFO
smartctl
lspci
lsblk
dmidecode
lscpu
lshw
hdparm

## 查找网卡的插槽地址

```
[root@wfpprddb01 tacy]#lspci -vvv|grep -i 'Ethernet' -A10

[root@wfpprddb01 tacy]# ethtool -i ens1f0
driver: ixgbe
version: 4.4.0-k
firmware-version: 0x800003df
expansion-rom-version:
bus-info: 0000:43:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no

```

# mouse & touchpad
升级系统系统的时候, 容易把synaptics的配置弄丢了, 注意修改

Natural scrolling

It is possible to enable natural scrolling through synaptics. Simply use negative values for VertScrollDelta and HorizScrollDelta like so:
/etc/X11/xorg.conf.d/50-synaptics.conf

```
Section "InputClass"
    Identifier "Trackpad"
    Driver "synaptics"
    MatchIsTouchpad "on"
    Option "HorizTwoFingerScroll" "1"
    Option "VertScrollDelta" "-111"
    Option "HorizScrollDelta" "-111"
    Option "HorizHysteresis" "72"
    Option "VertHysteresis" "72"
    Option "PalmDetect" "3"
    Option "PalmMinWidth" "12"
    Option "PalmMinZ" "20"
    Option "AreaRightEdge" "4461"
    Option "AreaTopEdge" "395"
    Option "FingerLow" "10"
    Option "FingerHigh" "50"
    MatchDevicePath "/dev/input/event*"
EndSection

```
缺省配置的触摸板, 两指右键容易触发滚动事件, 导致焦点错位, 需要设置HorizHysteresis和VertHysteresis两个值.

另外一个问题就是手掌误触发移动事件, 这个需要设置PalmDetect参数, 需要自己慢慢调节, 注意参数是PalmMinWidth, 我的机器需要调到8才行, 太小正常的手指移动有问题. 但是这个问题通过设置Palm依然不能解决, 最终通过设置AreaRightEdge和AreaTopEdge解决, 这个的意思就是屏蔽掉一部分触摸板, 这里把右边和上边的部分触摸板屏蔽了, 最终解决了这个鼠标乱跳的烦人问题

另外就是AreaLeftEdge在我电脑上是负数, 很奇怪, 无法设置(设置为1就把半边的触摸板屏蔽了), 本来打算把Left也设置以下, 防止右手的误触. 触摸板的四个值参考synaclient输出LeftEdge/RightEdge/TopEdge/BottomEdge.

还有一个问题就是错误的认为你释放了鼠标键(untouch), 这个问题需要调整FingerLow, 别让这个值太高, 因为移动鼠标的时候可能用力不均匀, 需要确保这个力度不会低于FingerLow, 否则触摸板错误的认为你释放了鼠标

没有解决的问题是两指触发的滚动事件, 垂直移动两指容易产生水平滚动事件, 反之亦然, 暂时没找到解决办法.

调整synaptics的值通过和synclient完成, 修改之后实时观察效果.


# aur
首先git clone代码, 然后`makepkg -sric`, s表示解决依赖, r表示移除编译依赖, i表示安装, c表示清楚编译后环境.

如果碰到`ERROR: One or more PGP signatures could not be verified!`, 使用`gpg --recv-keys`接受key, 也可以`makepkg --skippgpcheck`.

## cower
aur包管理器, 可以查看安装的aur包, 是否由更新, 下载更新, 然后通过makepkg安装更新

# mimetype
`xdg-mime default org.gnome.Nautilus.desktop inode/directory`


# tor
pacman安装即可, 然后需要修改以下/etc/tor/torrc里面的配置, 缺省没有打开controller, 如果你需要编程调用, 需要打开该端口, 同时修改hashpasswd, hashpasswd可以通过'tor --hash-password yourpassws'获取.

如果只需要脚本控制tor, 就不需要配置tor称为bridge relay或者relay, 容易引起麻烦. 如果只需要给自己提供bridge, 可以配置:

```
BridgeRelay 1
PublishServerDescriptor 0
```

# fonts #

The font paths initially known to Fontconfig are: /usr/share/fonts/, ~/.local/share/fonts (and ~/.fonts/, now deprecated). Fontconfig will scan these directories recursively. For ease of organization and installation, it is recommended to use these font paths when adding fonts.
To see a list of known Fontconfig fonts: `fc-list : file`

字体控制通过gnome-tweaks调整, 建议用12号字体

一般安装字体:

```
ttf-inconsolata                                 #程序员字体, 从验证结果看, 这个字体支持中英文混编效果最好(用12号字体)
adobe-source-han-sans-cn-fonts 1.004-1          #中文
```

字体配置文件在/etc/fonts目录下, 用户一般定义自己的配置在/etc/fonts/local.conf目录:

```
[tacy@tacyArch fonts]$ cat /etc/fonts/local.conf
<!DOCTYPE fontconfig SYSTEM "../fonts.dtd">
<fontconfig>
  <!-- Generic name aliasing -->
  <alias>
    <family>sans-serif</family>
    <prefer>
      <family>Source Han Sans CN</family>
    </prefer>
  </alias>
  <!-- Generic name aliasing -->
  <alias>
    <family>monospace</family>
    <prefer>
      <family>Inconsolata</family>
    </prefer>
  </alias>
</fontconfig>
```
配置完成之后, monospace优先使用Inconsolata, sans-serif/sans优先使用SourceHanSansCN

查看字体列表`fc-list : file`

查看匹配字体`fc-match sans -a`

# powerline #
sudo pip install powerline-status  #install into: /usr/bin/ & `pip show powerline-status`

## install powerline fonts #
```
cd  ~/usr/share/fonts/tacy
wget https://github.com/powerline/powerline/raw/develop/font/PowerlineSymbols.otf
fc-cache ./tacy
cd /etc/fonts/conf.avail
wget https://github.com/powerline/powerline/raw/develop/font/10-powerline-symbols.conf
cd /etc/fonts/conf.d
ln -s /etc/fonts/10-powerline-symbols.conf .
```

## powerline daemon by systemd ##
```
cat ~/.config/systemd/user/powerline-daemon.service
[Unit]
Description=powerline-daemon - Daemon that improves powerline performance
Documentation=man:powerline-daemon(1)
Documentation=https://powerline.readthedocs.org/en/latest/

[Service]
ExecStart=/usr/bin/powerline-daemon --foreground

[Install]
WantedBy=default.target

systemctl --user enable powerline-daemon
systemctl --user start powerline-daemon
```

## tmux ##
add `source /usr/lib/python3.5/site-packages/powerline/bindings/tmux/powerline.conf` to `~/.tmux.conf`

## bash ##
add `. /usr/lib/python3.5/site-packages/powerline/bindings/bash/powerline.sh` to `~/.bash_profile`

## dns
chattr +i /etc/resolv.conf

# tmux

```
# remap prefix to Control + a
unbind C-b
set -g prefix C-x
bind C-x send-prefix

unbind r
bind r source-file ~/.tmux.conf

set -g default-shell /bin/zsh

## copy-mode
unbind [
bind-key -T prefix e copy-mode
# move x clipboard into tmux paste buffer
# To copy:
bind-key -n -t emacs-copy M-w copy-pipe "xclip -i -sel p -f | xclip -i -sel c "
# To paste:
bind-key -n C-y run "xclip -o | tmux load-buffer - ; tmux paste-buffer"


## * Window Management
set -g base-index 1 # start window indices at 1
set -g renumber-windows on
```

# zsh #

## Install

> pacman -Ss zsh

## Config
我的缺省Shell依然是用的Bash, 通过Bash激活tmux, 然后设置tmux的缺省shell为zsh, 省得对系统做修改

1. 在`~/.tmux.conf`中加入: `set -g default-shell /bin/zsh`

2. 在`~/.bashrc`中判断是否有tmux实例, 如果没有创建一个新的, 否则attach它

``` sh
#
# ~/.bashrc
#

# If not running interactively, don't do anything
[[ $- != *i* ]] && return

alias ls='ls --color=auto'
PS1='[\u@\h \W]\$ '
export PATH=~/bin:$PATH

# tmux
if which tmux >/dev/null 2>&1; then
    ID="`tmux ls | grep -vm1 attached | cut -d: -f1`" # get the id of a deattached session
    if [[ -z "$ID" ]] ;then # if not available create a new one
        tmux new-session
    else
        tmux attach-session -t "$ID" # if available attach to it
    fi
fi
```

3. 通过systemd启动tmux
>systemctl --user start tmux

```
[Unit]
Description=Start tmux in detached session

[Service]
Type=forking
ExecStart=/usr/bin/tmux new-session -s %u -d
ExecStop=/usr/bin/tmux kill-session -t %u

[Install]
WantedBy=multi-user.target

```



## Oh-my-zsh

> sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

my custom conf:
```
# ********************************************************************
## tacy cust ##
HISTFILE=~/.histfile
HISTSIZE=1000
SAVEHIST=1000
setopt HIST_FIND_NO_DUPS HIST_IGNORE_ALL_DUPS HIST_IGNORE_DUPS HIST_IGNORE_SPACE HIST_SAVE_NO_DUPS

# Automatically included new bin file in the completion
zstyle ':completion:*' rehash true

# emacs mode
bindkey -e

# app set
## set relative app
evince_bg() { evince "$@" & }
alias -s pdf=evince_bg

emacs_bg() { emacsclient "$@" & }
alias -s go=emacs_bg -nc
alias -s html=emacs_bg -nc
alias -s py=emacs_bg -nc
alias -s md=emacs_bg -nc
alias -s go=emacs_bg -nc
# ********************************************************************
```

## autojump ##
> pacman -S autojump

`d`: 列出最近的目录列表
`j number`: 其中的number是`d`命令输出中的序号, 可以完成快速跳转.
`j kube`: 跳转到历史目录中保护kube的路径
`j -s`: 列出历史路径
`jo`: 打开文件浏览器

## Using
C-p / C-n      #history substring search

# network
## BBR
``` shell
cat /etc/sysctl.d/50-bbr.conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

lsmod|grep -i bbr
```
## mtu
ip link show |grep mtu
ip link set eth0 mtu 9000  # (JUMBO frames)
/etc/sysconfig/network-scripts/<interface>  add line 'MTU="9000"' or accroding to archlinux jumbo frames


## 网卡bonding
`teamdctl team0 state`


## software AP
最简单的办法就是使用[create_ap](https://github.com/oblique/create_ap), 一条命令搞定`create_ap wlan0 wlan0 MyAccessPoint MyPassPhrase`

具体参考[software access point](https://wiki.archlinux.org/index.php/software_access_point)
## ssh reverse resolv
There are several things that can go wrong. Add -vvv to make ssh print a detailed trace of what it's doing, and see where it's pausing.

The problem could be on the client or on the server.

A common problem on the server is if you're connecting from a client for which reverse DNS lookups time out. (A “reverse DNS lookup” means getting back from the client machine's IP address to a host name. It isn't really useful for security, only slightly helpful to diagnose breakin attempts from log entries, but the default configuration does it anyway.) To turn off reverse DNS lookups, add UseDNS no to /etc/ssh/sshd_config (you need to be root on the server; remember to restart the SSH service afterwards).

Another thing that can go wrong is GSSAPI authentication timing out. If you don't know what that is, you're probably not relying on it; you can turn it off by adding the line GSSAPIAuthentication no to /etc/ssh/ssh_config or ~/.ssh/config (that's on the client side).

## mitmproxy
证书地址`http://mitm.it`, android模拟器运行时加参数`-http-proxy=ip:8080`

# Qemu
## mount qcow2

```
$modprobe nbd max_part=16
$qemu-nbd --connection=/dev/nbd0 node_two.qcow2
$fdisk -l /dev/nbd0
Disk /dev/nbd0: 8 GiB, 8589934592 bytes, 16777216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000aa3f5

Device      Boot Start      End  Sectors Size Id Type
/dev/nbd0p1 *     2048 16777215 16775168   8G 83 Linux

$mount /dev/nbd0p1 tmp

```

有时候没办法mount, 比如盘元数据有问题, 你可以用工具修复以下, 比如:`xfs_repair -L /dev/nbd0p1`

## tftp server
dnsmasq内嵌tftp server, 直接修改/etc/dnsmasq配置, 指定下面选项即可:
```
tftp-root=/srv/tftp
enable-tftp
```
如果需要配置ubuntu的pxe server, 下载netboot.tar.gz包, 解压到/srv/tftp即可

注意网络里面的dhcpserver, 容易导致问题. 另外, 无线网卡需要支持ipxe才行

## software
### base-utils
`du -s ./* | sort -n` 查询磁盘空间

### imagemagick

```
find . -name '*-1.jpg' -exec mogrify -font Source-Han-Sans-CN-Heavy -fill '#E91E63' -draw 'rectangle 0,565,800,705' -pointsize 120 -gravity center -fill black -annotate +0+230 '日本直邮到手' -fill white -annotate +5+235 '日本直邮到手' '{}' \; -print

find . -name '*.JPG' -exec mogrify -filter Triangle -define jpeg:extent=45KB -thumbnail 800x800\! '{}' \;
```

### ffmpeg

```
find . -name '*.MOV' -exec ffmpeg -i '{}' -vcodec libx264 -crf 32 '{}'.mp4
```

### chrome
1. remote desktop -> app(应用) -> chrome remote desktop -> share(分享)
2. chrome的dns总是不走代理, 会导致问题, 如果实在很急, 可以用下面命令启动chrome: `google-chrome --proxy-server="socks5://localhost:8888"`
3. 清除chrome dns cache, 两步: 首先, `chrome://net-internals/#dns` -> `Clear host cache`, 其次, `chrome://net-internals/#sockets` ->`Flush socket pools“`, 后者可以观察到dns用的代理情况, 对于一些奇怪的问题诊断非常有用.



### bash
find finances -name '*.vue' -maxdepth 1 -exec wc -l '{}' \; |cut -d' ' -f1|paste -sd+ |bc


### eclipse
Disable GTK+ 3
When the SWT GTK+ 3 UI is buggy and sometimes unusable, You can try to disable the use of GTK+ 3 with add the following to /usr/lib/eclipse/eclipse.ini.
```
--launcher.GTK_version
2
```
Those two lines must be added before:

``` text
--launcher.appendVmargs
```


### pacman

pacman -Qo filename / pkgfile filename
pactree 可以查看包依赖图, 在pacman-contrib包里面
pacman -Sc 清除缓存包(系统里面没有安装的)

# mitmproxy

安装: `pacman -S mitmproxy`, 运行: `mitmweb`, 设置需要监控的程序走代理, 同时在浏览器打开监听网址, 就可以看到所有的web请求了

如果是ssl, 可能需要信任mitmproxy的证书, 你可以在你的设备的浏览器上打开`mitm.it`这个网址, 安装证书即可. 你也可以收到安装, 默认mitmproxy会生成证书在`$HOME/.mitmproxy/`目录下, 也可以在mitmproxy启动的时候指定证书

具体可以参考https://mitmproxy.org/doc/certinstall.html#docQuick

# DEBUG
https://wiki.archlinux.org/index.php/Debug_-_Getting_Traces
https://wiki.archlinux.org/index.php/Patching_packages
ABS/asp

ldd

# FS
## major/minor
通过lsblk可以查看major:minor
ls -l /dev 也可以查看major:minor
`brw-rw----  1 root disk     8,   0 Dec 19 07:59 sda`
最前面的b代表block设备

## debugfs（ext4/3 fs)
通过inode找文件：

``` shell
sudo debugfs /dev/sda6
debugfs: ncheck <inode number>
Inode Pathname
10403886  /home/tacy/i2p
```
## lsof
列出系统内打开的所有文件

## file lock
lslocks  -> utils-linux
/proc/locks

# Network
iproute2替换之前的大部分网络工具， 例如netstat/ifconfig/route等

以后常用的工具ip和ss

## ss
### 查看当前tcp状态
ss -top
可以查看当前所有的连接：包括状态/定时器（retrans/keepalive)，发送和接受队列，那个进程发起，用户是谁都能看到

``` shell
ss -o state established '( dport = :ssh or sport = :ssh )'
  Display all established ssh connections.

ss -x src /tmp/.X11-unix/*
  Find all local processes connected to X server.
```

## sockdump
这是别人用brf写的一个监控unix socket的小工具，可以查看unix socket间的通讯记录
用`ss -xlp`可以查看系统处于listen状态下的unix socket

## offload
ethtool -k ethY
ethtool -K ethY tso on

# Utils
## coreutils
这个工具集里面的所有工具你都看一遍man，基本上都是些很常用的工具，例如：cat/mkdir/cp/rm/tail/head/sort/nohup/mv啥的
## sysstat & procps-ng
这里面基本都是系统状态有关的工具，例如：top/vmstat/iostat/mpstat/ps/free/pidof/sysctl/pgrep/sar/pidstat
## iproute2
网络配置和状态查看相关工具：ip/ifcfg/bridge/ss/tc
## util-linux
大工具包，里面的工具包括：kill/fdisk/su/lslogins/lscpu/lsns/dmesg/nsenter/fsck/namei
## binutils
库相关工具，例如:ld/nm/strings/objdump/objcopy
## iptables
防火墙相关软件，主要是iptables
## systemd
服务相关工具： systemd/journalctl/coredumpctl/busctl/gdbus
## 其他
man / debugfs / gdb / lsof

## log
last -n5 -x reboot shutdown
ausearch -i -m system_boot,system_shutdown
ethtool interface 查网卡速度 / 设置offload
teamdctl team0 state

## rhel
### selinux
getenforce
# my server configuration
https://www.digitalocean.com/community/tutorials/how-to-install-dropbox-client-as-a-service-on-centos-7
https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-centos-7
