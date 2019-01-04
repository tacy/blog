---
title: "android notes"
date: 2014-07-29
lastmod: 2014-07-29
draft: false
tags: ["tech", "android", "notes"]
categories: ["tech"]
description: "android相关知识笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# Android

## emulator
为了模拟器性能[fn:1]，需选择x86的cpu，同时打开haxm（intel)支持，在ubuntu下打开hamx，
只需加载kvm模块，同时启动命令上添加‘-qemu -enable-kvm’ 参数即可，启动命令：

: emulator64-x86 -avd test -data userdata.img -qemu -m 2048 -enable-kvm

其中的data如果没有指定，emulator好像每次都会初始化数据，也就是说重启之后，通过
adb安装的应用就没有了。

如果启动出现如下信息：
: emulator: emulator window was out of view and was recentered
你可以添加'-scale auto'参数（取值范围0.1-3)，注意，非qemu参数都加在qemu前面。

另外avd文件存放在home/.android目录下。

你可以通过mksdcard创建sd镜像文件。

通过adb安装app和拷贝文件：
``` shell
adb push localfile /sdcard/
adb install test.apk
```

启动sdk manager：

: android sdk

启动avd manager：

: android avd

具体的其他子命令可以help，主要是abd和android两个。
# ENV
android目录在workspace下

# android sdk

About the version of Android SDK Build-tools, the answer is

By default, the Android SDK uses the most recent downloaded version of the Build Tools.

The [build] tools, such as aidl, aapt, dexdump, and dx, are typically called by the Android build tools or Android Development Tools (ADT), so you rarely need to invoke these tools directly. As a general rule, you should rely on the build tools or the ADT plugin to call them as needed.
Source

Anyway, here is a synthesis of the differences between tools, platform-tools and build-tools:

Android SDK Tools
Location: $ANDROID_HOME/tools
Main tools: ant scripts (to build your APKs) and ddms (for debugging)
Android SDK Platform-tools
Location: $ANDROID_HOME/platform-tools
Main tool: adb (to manage the state of an emulator or an Android device)
Android SDK Build-tools
Location: $ANDROID_HOME/build-tools/$VERSION/
Documentation
Main tools: aapt (to generate R.java and unaligned, unsigned APKs), dx (to convert Java bytecode to Dalvik bytecode), and zipalign (to optimize your APKs)

# reverse engineering
编译过程是java->smali->dex, smali负责smali->dex, baksmali负责dex->smali, smali类汇编, 但是也可以修改.

一般如果你需要先获取apk, 然后通过apktool, 获取到apk的资源文件和smali, 然后可以直接通过smaliide(android studio插件)调试, 具体参考[smalidea](https://github.com/JesusFreke/smali/wiki/smalidea)

你也能通过baksmali获取到smali代码, 修改之后smali重新打包

如果觉得smali很难懂, 也可以通过dex2jar把dex转换成jar包, 然后通过java反编译工具获取java代码, 当然这个反编译一般都是不完全的, 但是你也能完成apk的修改, 然后重新打包

# emulator
## set proxy
Go to Settings>WIRELESS & NETWORKS>Mobile networks>Mobile networks settings>Access Point Names
add access point by clicking "+" button and then type favorite name for Name, favorite name for APN, fixed Proxy and port. If necessary put user name and password.

## fake gps location
直接通过模拟器发送就行了


# android studio
启动bin/studio.sh
配置文件玩在~/.AndroidStudio1.5目录下, 调整jvm参数在该目录下的studio.vmoptions,

配置sdk: file->setting
配置project sdk: file->project structure下, platform settings -> sdks中选择"+". 还需要配置project运行用的sdk, project setting -> project中选择project sdk和project language level

avd管理: tools -> android -> avd manager


## debug ##

首先`lsusb`获取到id, 例如`us 001 Device 004: ID 17ef:776d Lenovo`

添加udev rules:

```
cat /etc/udev/rules.d/51-android.rules
DB your devices
SUBSYSTEM=="usb", ATTR{idVendor}=="17ef", ATTR{Product}=="776d", MODE="0666"
```

然后执行`sudo adb kill-server && sudo adb start-server`

接下来执行`adb devices`, 应该看到下面这样的行输出:

```
List of devices attached
813d4882        device
```

### 获取手机上的应用

``` abap
adb shell pm list packages -f
adb pull -p PATH/base.apk OUTPUT.apk

安装到指定设备
adb -s "deviceIDfromlist" install path+apkName
```

### apk
查看apk适配的架构

``` abap
/apk/# build-tools/version/aapt dump badging ymatou-shop.apk
native-code: 'arm64-v8a' 'armeabi' 'armeabi-v7a' 'x86' 'x86_64'
```


## 字体显示方框 ##

setting -> Appearance -> Override default fonts by (not recommended)

## migrate eclipse project to android studio ##

svn编码是gbk, 所以checkout代码出来的时候需要先修改LANG=zh_CN.gbk
svn checkout file:///data/svn/android/zhijie/branches/vTest .
svn checkout file:///data/svn/android/zhijie/branches/abs-library abs-library
svn checkout file:///data/svn/android/zhijie/branches/sliding-library sliding-library

然后android studio直接import eclipse项目

接下来抛错误:
Error:(12, 0) Error: NDK integration is deprecated in the current plugin.  Consider trying the new experimental plugin.  For details, see http://tools.android.com/tech-docs/new-build-system/gradle-experimental.  Set "android.useDeprecatedNdk=true" in gradle.properties to continue using the current NDK integration.
<a href="openFile:/home/tacy/workspace/android/zhijie_pay/zhijie_pay/build.gradle">Open File</a>

修改gradle.properties, 增加行`android.useDeprecatedNdk=true`

继续:
ERROR: 9-patch image /home/tacy/workspace/android/zhijie_pay/zhijie_pay/src/main/res/drawable/navbar.9.png malformed

直接修改文件名为navbar.png(临时改法)

undefined reference to `__android_log_print'
http://stackoverflow.com/questions/4455941/undefined-reference-to-android-log-print

Error:(63, 14) error: cannot find symbol method addOnPageChangeListener(<anonymous OnPageChangeListener>)
http://stackoverflow.com/questions/31099555/cannot-find-symbol-method-addonpagechangelistener



01-31 15:43:12.995 9628-9628/? E/dalvikvm: dlopen("/data/app-lib/com.zhijie.android-1/libBaiduMapSDK_v2_3_1.so") failed: Cannot load library: load_library(linker.cpp:761): not a valid ELF executable: /data/app-lib/com.zhijie.android-1/libBaiduMapSDK_v2_3_1.so


emulator: ERROR: This AVD's configuration is missing a kernel file!!
需要安装对应的image



#zhijie
java/com/zhijie/android/login/LoginActivity
java/com/zhijie/android/shopping/net/HttpManager

client:13959203004 | e47ca7a09cf6781e29634502345930a7 | 4e5ceed89a82a06a37e2bd0f0d8f49a4

00000000 md5 => dd4b21e9ef71e1291183a46b913ae6f2
device_id => 914ddf9d048008cb58390d20a94d9d05
username => 13959203004


client:18642716660 | e10adc3949ba59abbe56e057f20f883e | 609f7d6df5c040d964aeff06ef57beba

deviceId=a76fe41f91946c4f96fde0fa4341e679&username=18642716600&password=d4f74be5b3c0a379fe3f12b6503a89ed&macAddress=34%3Ac3%3Ad2%3A06%3Ad3%3A32&registTime=1454570913373HTTP/1.1 200 OK

update client set password='dd4b21e9ef71e1291183a46b913ae6f2', deviceid='914ddf9d048008cb58390d20a94d9d05' where username='18642716660';

device_info: terminal   | tm38d628d1f73b896e77ace749a3301923 |

update device_info set devicetype='pad', deviceid='914ddf9d048008cb58390d20a94d9d05' where id='360';

select o.cid from client c join device_info o on c.cid=o.cid where o.deviceid='914ddf9d048008cb58390d20a94d9d05' and c.username='18642716660' and c.password='dd4b21e9ef71e1291183a46b913ae6f2';


# 常用
## yarn
yarn add -S packagename / yarn add -D packagename / yarn remove packagename

## react native
react-native link  packagename /  react-native unlink packagename

### FAQ

1. Unable to install /home/admin/MyApp/app/build/outputs/apk/app-local-debug.apk

Step 1: Make sure your device is connected and then run adb devices. From the output, grab the device id H80xxxxxx.
Step 2: Run react-native run-android --deviceId H80xxxxxx

# Footnotes

[fn:1] [[http://stackoverflow.com/questions/1554099/why-is-the-android-emulator-so-slow][Why is the Android emulator so slow?]]
