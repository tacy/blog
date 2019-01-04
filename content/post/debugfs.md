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
debugfs Command Examples

# Use debufs to prowl around a file system.

> debugfs /dev/hda6
debugfs 1.19, 13-Jul-2000 for EXT2 FS 0.5b, 95/08/09

# list files

debugfs:  ls
2790777 (12) .   32641 (12) ..   2790778 (12) dir1   2790781 (16) file1
2790782 (4044) file2

#  List the files with a long listing

#  Format is:
# Field 1:  Inode number.
# Field 2:  First one or two digits is the type of node:
#    2 = Character device
#    4 = Directory
#    6 = Block device
#    10 = Regular file
#    12 = Symbolic link
#
#    The Last four digits are the Linux permissions
# 3. Owner uid
# 4. Group gid
# 5. Size in bytes.
# 6. Date
# 7. Time of last creation.
# 8. Filename.

debugfs:  ls -l
2790777  40700   2605   2601    4096  5-Nov-2001 15:30 .
 32641   40755   2605   2601    4096  5-Nov-2001 14:25 ..
2790778  40700   2605   2601    4096  5-Nov-2001 12:43 dir1
2790781 100600   2605   2601      14  5-Nov-2001 15:29 file1
2790782 100600   2605   2601      14  5-Nov-2001 15:30 file2

# dump the contents of file1

debugfs: cat file1
This is file1

# dump an inode to a file (same as cat, but to a file) and using
#  instead of the file name.

debugfs: dump <2790782> file1-debugfs

# dump the contents of an inode

debugfs: stat file1
Inode: 2790782   Type: regular    Mode:  0600   Flags: 0x0   Generation: 46520506
User:  2605   Group:  2601   Size: 14
File ACL: 0    Directory ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
ctime: 0x3be712ea -- Mon Nov  5 15:30:02 2001
atime: 0x3be712ea -- Mon Nov  5 15:30:02 2001
mtime: 0x3be712ea -- Mon Nov  5 15:30:02 2001
BLOCKS:
5603924
TOTAL: 1

# Dump an directory inode and look at it.

debugfs: dump dir1 dir1-debugfs

# Leave debugfs or use another xterm to look at the contents
# using od or xxd.  The format of a directory (ext2 version 2.0) is:

# Field 1. Four byte inode number.
# Field 2. Two byte directory entry length.
# Field 3. Two byte file name length.
# Field 5. Filename (1-255 characters).
# Pad.     The filename is padded to be a multiple of 4 bytes long.


# use -c to see the file names and single byte values
# You can see the file names and identify the locatin of
# the other fields.  Of importance, the length of the
# entries (octal); . (4-5), .. (20-21), file3 (34-35),
# file4 (54-55), .file4.swp (74-75),  ...

> od -c dir1-dump
0000000   z 225   *  \0  \f  \0 001 002   .  \0  \0  \0   y 225   *  \0
0000020  \f  \0 002 002   .   .  \0  \0 202 225   *  \0 020  \0 005 001
0000040   f   i   l   e   3  \0  \0  \0 201 225   *  \0   � 017 005 001
0000060   f   i   l   e   4  \0  \0  \0 177 225   *  \0   � 017  \n 001
0000100   .   f   i   l   e   4   .   s   w   p  \0  \0 200 225   *  \0
0000120   � 017 006 001   f   i   l   e   4   ~   .   s   w   p   x  \0
0000140  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
*
0010000

# use -d to see the two byte values

> od -d dir1-dump
0000000 38266    42    12   513    46     0 38265    42
0000020    12   514 11822     0 38274    42    16   261
0000040 26982 25964    51     0 38273    42  4056   261
0000060 26982 25964    52     0 38271    42  4040   266
0000100 26158 27753 13413 29486 28791     0 38272    42
0000120  4020   262 26982 25964 32308 29486 28791   120
0000140     0     0     0     0     0     0     0     0
*
0010000

# You can see that the lengths of the entries are:
#    . = 12, .. = 12, file3 = 16, file4 = 4096
# Whoa! what happened there.  The file .file4.swp
# and any other files in the directory have been deleted,
# so the length of the entry goes to the end of the block
#
# use -l to see the four byte values.  We can see the inode
# values of the files.

> od -l dir1-dump
0000000     2790778    33619980          46     2790777
0000020    33685516       11822     2790786    17104912
0000040  1701603686          51     2790785    17108952
0000060  1701603686          52     2790783    17436616
0000100  1818846766  1932407909       28791     2790784
0000120    17174452  1701603686  1932426804     7893111
0000140           0           0           0           0
*
0010000


-------------------------------------------------------------------<



#
# You inadvertently delete a file you want back.  The file was named
# /home/harkin/test/file2.  Immediately do the following.
#


> umount /home

# so that you don't create a new file that overwrites the inode
# or use one of the file blocks.

# Execute df to find out what partition /home is on

>df
Filesystem           1k-blocks      Used Available Use% Mounted on
/dev/hda1              1011928    507860    452664  53% /
/dev/hda8             27364092   1890176  24083896   8% /home
/dev/hda5              8064272   3492760   4161860  46% /usr
/dev/hda7              1011928     87956    872568  10% /var
clowns:/db/boze       17783240  10494056   7183568  60% /home/bozo/db

# Get the data on the /home filesystem

tune2fs -l /dev/hda8 | grep "Block size"

   Block size:               4096

# So the block size is 4096 bytes.

# Create a file system to duplicate the /home file system in case
# you screw up royally.  This disk should be exactly the same size
# as the file system you are backing up.  Fortunately there is an
# unused disk /dev/hdb.

> fdisk /dev/hdb
Command (m for help): n
Command action
   l   logical (5 or over)
   p   primary partition (1-4)
p

+27364092K
w

# copy /home to the backup location

dd if=/dev/hda8 of=/dev/hdb1 bs=4096

# Now use debugfs to try to fix things.  We need to try to
# find the inode of the deleted file.  Use lsdel to
# list all of the deleted inodes on the file system.

debugfs -w            # to allow writing
debugfs:  lsdel
3061 deleted inodes found.
 Inode  Owner  Mode    Size    Blocks    Time deleted
                     .
                     .
3296723   2605 100600    652    1/   1 Fri Nov  2 07:30:33 2001
3296724   2605 100600   1545    1/   1 Fri Nov  2 07:30:33 2001
3296725   2605 100600    355    1/   1 Fri Nov  2 07:30:33 2001
3296731   2605 100600    440    1/   1 Fri Nov  2 07:30:33 2001
3296732   2605 100600   3536    1/   1 Fri Nov  2 07:30:33 2001
3296733   2605 100600   2365    1/   1 Fri Nov  2 07:30:33 2001
3296734   2605 100600    443    1/   1 Fri Nov  2 07:30:33 2001
3296850   2605 100600   2046    1/   1 Fri Nov  2 07:30:33 2001
3296851   2605 100600    729    1/   1 Fri Nov  2 07:30:33 2001
3296852   2605 100600    850    1/   1 Fri Nov  2 07:30:33 2001
3296853   2605 100600   3251    1/   1 Fri Nov  2 07:30:33 2001
3296854   2605 100600   3733    1/   1 Fri Nov  2 07:30:33 2001
3296855   2605 100600   3109    1/   1 Fri Nov  2 07:30:33 2001
3296856   2605 100600   3211    1/   1 Fri Nov  2 07:30:33 2001
652818   2605 100600 171791   43/  43 Fri Nov  2 16:07:33 2001
897613   2605 100600   2096    1/   1 Mon Nov  5 07:49:28 2001
979218   2605 100600   3797    1/   1 Mon Nov  5 07:49:29 2001
979219   2605 100600   4096    1/   1 Mon Nov  5 07:49:29 2001
179573   2605 100600   9113    3/   3 Mon Nov  5 12:41:16 2001
636513   2605 100600   1327    1/   1 Mon Nov  5 12:41:16 2001
636520   2605 100600     20    1/   1 Mon Nov  5 12:41:16 2001
1338319   2605 100600   6998    2/   2 Mon Nov  5 12:48:55 2001

# Based on the time and date, the inode to restore is 179573, 636513
# or 636520.  Try to figure out which one.

debugfs:cat <179573>
   .
   .

debugfs:cat <636513>
   .

# This is rather inconvenient.  If the directory where the files were
# deleted from still exists, use the cd command to get there and then
# use ls -d  which lists the files in the directory only, including
# those with the deleted flag set.

1566721  (12) .    32641  (12) ..    1566788  (60) 530
1566790 (48) file1   1566791 (24) file2
<1566747>  (20) file3

The inode numbers in brackets are deleted files.  A better looking display
comes with ls -ld.


# So now you know which inode you need to restore.
# To restore the file, you need to modify the inode, not the
# directory entry.  This can be done with the modify_inode (mi)
# command.  Specifically, change the deletion time to zero
# and the link count to 1.

debugfs: mi <636513>
debugfs:  mi <148003>
                              Mode    [0100644]
                           User ID    [510]
                          Group ID    [510]
                              Size    [8123]
                     Creation time    [904216575]
                 Modification time    [904234782]
                       Access time    [904234782]
                     Deletion time    [904236721] 0
                        Link count    [0] 1
                       Block count    [16]
                        File flags    [0x0]
                         Reserved1    [0]
                          File acl    [0]
                     Directory acl    [0]
                  Fragment address    [0]
                   Fragment number    [0]
                     Fragment size    [0]
                   Direct Block #0    [100321]
                   Direct Block #1    [100322]
                   Direct Block #2    [100323]
                   Direct Block #3    [100324]
                   Direct Block #4    [200456]
                   Direct Block #5    [200457]
                   Direct Block #6    [200675]
                   Direct Block #7    [200675]
                   Direct Block #8    [304568]
                   Direct Block #9    [0]
                  Direct Block #10    [0]
                  Direct Block #11    [0]
                    Indirect Block    [0]
             Double Indirect Block    [0]
             Triple Indirect Block    [0]

# It has been recovered.

# This won't work for files with indirect blocks and you might find that
# one or more blocks have been reused already.  If so, you can
# recover as much data as possible by dumping the blocks to a file.

debugfs: dump <100321> /tmp > file1.000
debugfs: dump <100322> /tmp >> file1.000

# and so on. For files that are longer than 12 blocks, you have to
# trace the indirect, double-indirect and triple-indirect blocks.
