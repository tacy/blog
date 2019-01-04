---
title: "Xmodmap配置介绍"
date: 2013-02-08
lastmod: 2013-02-08
draft: false
tags: ["tech", "ubuntu", "emacs"]
categories: ["tech"]
description: "了解xmodmap纯粹是为了解决emacs的问题，这里是过程中的一些记录，包括xmodmap的简单介绍以及如何通过xmodmap重新映射键盘键的位置的配置方法，方便自己查看，如果你有同样问题，希望能对你有所帮助。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# introduce
xmodmap能够重新映射键盘上键的布局，通常情况下，你不会需要对键盘做重新映射，只有在特定情况下，比如emacs，需要频繁操作Ctrl键，但是偏偏US标准键盘上，Ctrl键的位置就很不合理，通常我们都会切换Caps_Lock和Ctrl_L的位置（以前的UNIX键盘Ctrl_R的位置就是在A键的旁边），达到方便操作的目的，下面文字主要是记录我自己的映射信息
# 一些特殊键的标准位置
下面是我的ubuntu上几个特殊键的位置，包括shift，ctrl，alt，super/win，menu键：

``` shell
keycode 133 = Super_L NoSymbol Super_L$
keycode 134 = Super_R NoSymbol Super_R
keycode 135 = Menu NoSymbol Menu
keycode  50 = Shift_L NoSymbol Shift_L
keycode  62 = Shift_R NoSymbol Shift_R
keycode  66 = Caps_Lock NoSymbol Caps_Lock
keycode  37 = Control_L NoSymbol Control_L
keycode  105 = Control_R NoSymbol Control_R
```


你可以通过如下命令获取：
`$xmodmap -pke > xmodmap.out`

# 我期望的效果
- 切换CapsLock和Ctrl-L
- 映射menu键为Ctrl-R
- 映射Ctrl-R键为Shift-R
- 取消menu键（当然你也可以把Shift-R键映射为menu，我嫌这个键容易错误触发中断工作，所以取消）
# 设置xmodmap的方法
你只需要在你的home目录下建立一个.Xmodmap的文件，在这个文件里面完成你的映射即可，无须其他操作（我的ubuntu是12.04）。重新映射键的三个主要步骤：
## 取消原先键映射
这里通过clear语法实现，下面是一些特殊键的表达[fn:1]

``` shell

  clear Shift
  clear Lock
  clear Control
  clear Mod1
  clear Mod2
  clear Mod3
  clear Mod4
  clear Mod5
```

我主要关注shift、lock、control、mod1和mod4，其中mod1表示alt，mod4表示super，这里我只需要取消下面几个：

``` shell
  clear Shift
  clear Lock
  clear Control
  clear Mod4
```

## 设定新的映射关系

``` shell
  keycode  66 = Control_L
  keycode  37 = Caps_Lock
  keycode 133 = Super_L
  keycode 134 = Control_R
  keycode 135 = Control_R
  keycode 50  = Shift_L
  keycode 62  = Shift_R
  !keycode 62  = Menu
  keycode 105 = Shift_R
```

## 绑定映射关系

``` shell
  !add mod1    = Alt_R Alt_L Meta_R Meta_L
  add Shift   = Shift_R Shift_L
  add Lock    = Caps_Lock
  add Control = Control_R Control_L
  add Mod4    = Super_R Super_L Menu
```

完成之后，保存你的xmodmap文件，可以通过如下命令立即生效：

`$xmodmap ~/.Xmodmap`

[fn:1]archlinux xmodmap setting https://wiki.archlinux.org/index.php/Xmodmap
