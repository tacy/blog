---
title: "FFmpeg使用笔记"
date: 2014-04-14
lastmod: 2014-04-14
draft: false
tags: ["tech", "linux", "multimedia"]
categories: ["tech"]
description: "这两天给孩子整理照片和视频，发现很多视频都非常大，一个3分钟的视频就300M，用vlc看了一下发现视频都没有做压缩，于是有了这个文章。"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# FFmpeg
视频压缩首先想到的自然是FFmpeg，我用的是Ubuntu，很不幸Ubuntu用Libav提到了FFmpeg
，具体为啥有了Libav建议libav官网八卦去，反正就是一些开发人员对项目管理者不满另起
炉灶啥的，这个暂且不提了[fn:6]。

关键是，Libav的资料太少了，去官网看了一下WIKI，直接就没兴趣用了，Google上搜索也
没有太多使用经验，虽然和FFmpeg大致相同，但总觉的别扭。我想把视频转成X264加AAC格
式，采用VBR方式（就是动态码率，更好的压缩比和视频质量），看了一下X264没问题，但
是AAC有好几个实现，其中最好的libfdk_aac不是GPL的license，所以Libav提供的版本不支
持，也就是说我需要自己编译[fn:1]。关键是这玩意编译还依赖很多外部包，人家FFmpeg编
译文档写的很清晰，Libav啥都没有。没办法，直接干掉，换FFmpeg来[fn:2]。


# 转码压缩
发现没啥可写的[fn:3][fn:4][fn:5]，别人都写的太清楚了，这里直接上我的命令吧：

: ffmpeg -i IMG_2596.MOV -c:v libx264 -crf 28 -c:a libfdk_aac -vbr 4 output.mp4

最后压缩下来，一个7分42秒的视频大概114M左右，之前可是650M，播放了一下，看不出啥
区别，效果不错，就是FFmpeg貌似不是那么高效，CPU都在nice状态，找个时间分析一下。

这里唯一需要解释一下的就是crf参数，该参数取值范围在0～51，值越低质量越高，当然
文件也越大，缺省值是23，一般18～28之间是比较合适的值，太低文件太大，太高图像质
量不高。该值±6对文件大小的影响基本上是一倍关系，+6文件大小缩小一半，-6文件大小
增加一倍[fn:7]。

修正一下，用下面命令压缩更快（CPU利用率更高了），而且压缩比好像更高，压缩之后只
有91M：

: ffmpeg -i IMG_2596.MOV -c:v libx264 -crf 28 -preset:v veryfast -c:a libfdk_aac -vbr 4 output.mp4

两者结果对比：

命令一：
``` shell
Stream mapping:
  Stream #0:0 -> #0:0 (h264 -> libx264)
  Stream #0:1 -> #0:1 (aac -> libfdk_aac)
Press [q] to stop, [?] for help
frame=11098 fps= 23 q=-1.0 Lsize=  111968kB time=00:07:42.33 bitrate=1983.9kbits/s dup=0 drop=5
video:107236kB audio:4392kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.304443%
[libx264 @ 0x27d10c0] frame I:70    Avg QP:27.89  size: 48280
[libx264 @ 0x27d10c0] frame P:6228  Avg QP:30.50  size: 13988
[libx264 @ 0x27d10c0] frame B:4800  Avg QP:32.41  size:  4024
[libx264 @ 0x27d10c0] consecutive B-frames: 13.5% 86.4%  0.1%  0.0%
[libx264 @ 0x27d10c0] mb I  I16..4: 15.7% 59.1% 25.2%
[libx264 @ 0x27d10c0] mb P  I16..4:  0.9%  1.8%  0.4%  P16..4: 61.4%  9.1%  8.4%  0.0%  0.0%    skip:18.1%
[libx264 @ 0x27d10c0] mb B  I16..4:  0.0%  0.0%  0.0%  B16..8: 48.7%  1.1%  0.1%  direct: 1.8%  skip:48.2%  L0:36.7% L1:62.5% BI: 0.8%
[libx264 @ 0x27d10c0] 8x8 transform intra:59.8% inter:73.3%
[libx264 @ 0x27d10c0] coded y,uvDC,uvAC intra: 58.3% 87.3% 46.3% inter: 15.6% 43.4% 2.3%
[libx264 @ 0x27d10c0] i16 v,h,dc,p: 18% 20% 19% 43%
[libx264 @ 0x27d10c0] i8 v,h,dc,ddl,ddr,vr,hd,vl,hu: 11% 12% 27%  8% 10%  8% 10%  7%  8%
[libx264 @ 0x27d10c0] i4 v,h,dc,ddl,ddr,vr,hd,vl,hu: 15% 18% 27%  7%  8%  6%  7%  5%  6%
[libx264 @ 0x27d10c0] i8c dc,h,v,p: 56% 18% 14% 11%
[libx264 @ 0x27d10c0] Weighted P-Frames: Y:0.2% UV:0.1%
[libx264 @ 0x27d10c0] ref P L0: 65.0% 14.6% 13.7%  6.8%  0.0%
[libx264 @ 0x27d10c0] ref B L0: 91.7%  8.3%  0.0%
[libx264 @ 0x27d10c0] ref B L1: 100.0%  0.0%
[libx264 @ 0x27d10c0] kb/s:1899.73
```

命令二：
``` shell
Stream mapping:
  Stream #0:0 -> #0:0 (h264 -> libx264)
  Stream #0:1 -> #0:1 (aac -> libfdk_aac)
Press [q] to stop, [?] for help
frame=11098 fps= 68 q=-1.0 Lsize=   89166kB time=00:07:42.33 bitrate=1579.9kbits/s dup=0 drop=5
video:84434kB audio:4392kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.382292%
[libx264 @ 0x367f940] frame I:69    Avg QP:27.91  size: 44978
[libx264 @ 0x367f940] frame P:6244  Avg QP:30.60  size: 11357
[libx264 @ 0x367f940] frame B:4785  Avg QP:32.14  size:  2600
[libx264 @ 0x367f940] consecutive B-frames: 13.8% 86.1%  0.1%  0.0%
[libx264 @ 0x367f940] mb I  I16..4: 45.3% 31.8% 23.0%
[libx264 @ 0x367f940] mb P  I16..4: 16.9%  2.9%  0.1%  P16..4: 38.6% 11.3%  3.8%  0.0%  0.0%    skip:26.3%
[libx264 @ 0x367f940] mb B  I16..4:  0.7%  0.1%  0.0%  B16..8:  9.9%  1.8%  0.1%  direct:17.4%  skip:70.0%  L0:36.8% L1:53.3% BI: 9.9%
[libx264 @ 0x367f940] 8x8 transform intra:15.2% inter:10.3%
[libx264 @ 0x367f940] coded y,uvDC,uvAC intra: 18.6% 73.0% 27.4% inter: 7.1% 29.0% 0.6%
[libx264 @ 0x367f940] i16 v,h,dc,p: 41% 26% 26%  8%
[libx264 @ 0x367f940] i8 v,h,dc,ddl,ddr,vr,hd,vl,hu: 10% 15% 47%  4%  5%  4%  5%  4%  5%
[libx264 @ 0x367f940] i4 v,h,dc,ddl,ddr,vr,hd,vl,hu: 15% 20% 25%  8%  8%  6%  7%  5%  7%
[libx264 @ 0x367f940] i8c dc,h,v,p: 59% 19% 17%  5%
[libx264 @ 0x367f940] Weighted P-Frames: Y:0.2% UV:0.1%
[libx264 @ 0x367f940] kb/s:1495.79
```

批量处理目录下的视频：

: for i in $(ls *.MOV); do ffmpeg -i $i -c:v libx264 -crf 28 -preset:v veryfast -c:a libfdk_aac -vbr 4 $i.mp4; done


# 旋转视频
有时候你的视频排的时候弄反了，看起来很麻烦，你可以通过FFmpeg旋转你的视频[fn:8]，
我需要垂直旋转我的视频，命令如下：
: fffmpeg -i IMG_0758.MOV -c:v libx264 -crf 28 -preset:v veryfast -c:a libfdk_aac -vbr 4 -vf "vflip" IMG_0758.MOV.mp4
垂直选择的参数是hflip，其他的90度旋转参考注脚8即可。


# 结尾
总之，FFmpeg是一个非常强大的开源视频处理工具，你能做各种事情（参考ffmpeg-filters
手册），使用它能对你的视频做各种处理，比如去logo，添加字幕，声轨，甚至合并声轨、
裁剪尺寸等，等待你去挖掘了。

最终我成功把我40G的视频压缩到5G，效果明显。

# Footnotes

[fn:1] [[http://unix.stackexchange.com/questions/77687/how-do-i-convert-flac-files-to-aac-preferrably-vbr-320k][How do I convert FLAC files to AAC]]

[fn:2] [[https://trac.ffmpeg.org/wiki/UbuntuCompilationGuide][Compile FFmpeg on Ubuntu, Debian, or Mint]]

[fn:3] [[http://slhck.info/video-encoding][Video Encoding]]

[fn:4] [[http://trac.ffmpeg.org/wiki/AACEncodingGuide][FFmpeg and AAC Encoding Guide]]

[fn:5] [[http://trac.ffmpeg.org/wiki/x264EncodingGuide][FFmpeg and x264 Encoding Guide]]

[fn:6] [[http://superuser.com/questions/507386/libav-vs-ffmpeg-better-to-use-libav-avconv-today][libav vs ffmpeg - better to use libav (avconv) today]]

[fn:7] [[http://slhck.info/articles/crf][CRF Guide]]

[fn:8] [[http://stackoverflow.com/questions/3937387/rotating-videos-with-ffmpeg][Rotating videos with FFmpeg]]
