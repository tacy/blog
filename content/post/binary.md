---
title: "binary notes"
date: 2019-01-05
lastmod: 2019-01-05
draft: false
tags: ["tech", "debug", "gdb", "linux", "tools"]
categories: ["tech"]
description: "关于二进制文件的笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

编译之后产生的二进制文件，有很多种不同的格式，一般在linux下，elf是最常用的，如果没有特殊说明，这里指的都是elf格式的二进制文件

# ELF
## readelf

* To retrieve a section header table:`readelf -S <object>`
* To retrieve a program header table: `readelf -l <object>`
* To retrieve a symbol table: `readelf -s <object>`
* To retrieve the ELF file header data: `readelf -e <object>`
* To retrieve relocation entries: `readelf -r <object>`
* To retrieve a dynamic segment: `readelf -d <object>`

# little endian & big endian
little endian： 9A9B9C9D   =>   (9D->0x1000, 9C->0x1001, 9B->0x1002, 9A->0x1003)
big endian则相反，存储成 （9A->0x1000, 9B->0x1001, 9C->0x1002, 9D->0x1003)

x86架构用的是little endian

# stack frame
## x86[^1]
x86在调用的时候，参数不会入register，都是压在栈里面，callee直接通过esp偏移找参数，简单代码示例：

```
void
bar(int a, int b)
{
    int x, y;

    x = 555;
    y = a+b;
}

void
foo(void) {
    bar(111,222);
}
```
foo被调用的时候，会把111，222存入到当前自己的stack frame，注意是从右到做入栈，所以他会做如下操作：

``` shell
esp = 当前esp - 8
222 存入esp-4
111 存入esp-8
开始调用bar，之前先把返回地址存入栈 (4bytes)
```
接下来，调用bar：

``` shell
保存foo的ebp (4bytes)
把esp塞入ebp，目前esp在的位置就是bar的ebp
esp = 当前esp - 16
555 存入当前esp - 4
取出a的值从(%ebp)+8, 放入%eax
取出b的值从(%ebp)+12,放入%edx
求和放入ebp-8
弹出保存的ebp，放入%ebp，重设esp为返回地址
```

## x64[^2]
看看下面代码，基本上你就大概知道x64是使用寄存器和栈一起在函数调用之间传递参数：

``` c++
int foobar(int a, int b, int c, int d, int e, int f, int g)
{
    int aa = a + 1;
    int bb = b + 2;
    int cc = c + 3;
    int dd = d + 4;
    int ee = e + 5;
    int ff = f + 6;
    int gg = g + 6;
    int sum = aa + bb + cc + dd + ee + ff + gg;

    return aa * bb * cc * dd * ee * ff * gg + sum;
}

int main()
{
    int a = foobar(33, 44, 55, 66, 77, 88, 99);
    return a + 1;
}
```

汇编代码`gcc -fno-asynchronous-unwind-tables -masm=intel -S simple.c -o simple.s`
``` assembly
        .file   "simple.c"
        .intel_syntax noprefix
        .text
        .globl  foobar
        .type   foobar, @function
foobar:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR -36[rbp], edi
        mov     DWORD PTR -40[rbp], esi
        mov     DWORD PTR -44[rbp], edx
        mov     DWORD PTR -48[rbp], ecx
        mov     DWORD PTR -52[rbp], r8d
        mov     DWORD PTR -56[rbp], r9d
        mov     eax, DWORD PTR -36[rbp]
        add     eax, 1
        mov     DWORD PTR -32[rbp], eax
        mov     eax, DWORD PTR -40[rbp]
        add     eax, 2
        mov     DWORD PTR -28[rbp], eax
        mov     eax, DWORD PTR -44[rbp]
        add     eax, 3
        mov     DWORD PTR -24[rbp], eax
        mov     eax, DWORD PTR -48[rbp]
        add     eax, 4
        mov     DWORD PTR -20[rbp], eax
        mov     eax, DWORD PTR -52[rbp]
        add     eax, 5
        mov     DWORD PTR -16[rbp], eax
        mov     eax, DWORD PTR -56[rbp]
        add     eax, 6
        mov     DWORD PTR -12[rbp], eax
        mov     eax, DWORD PTR 16[rbp]
        add     eax, 6
        mov     DWORD PTR -8[rbp], eax
        mov     edx, DWORD PTR -32[rbp]
        mov     eax, DWORD PTR -28[rbp]
        add     edx, eax
        mov     eax, DWORD PTR -24[rbp]
        add     edx, eaxn
        mov     eax, DWORD PTR -20[rbp]
        add     edx, eax
        mov     eax, DWORD PTR -16[rbp]
        add     edx, eax
        mov     eax, DWORD PTR -12[rbp]
        add     edx, eax
        mov     eax, DWORD PTR -8[rbp]
        add     eax, edx
        mov     DWORD PTR -4[rbp], eax
        mov     eax, DWORD PTR -32[rbp]
        imul    eax, DWORD PTR -28[rbp]
        imul    eax, DWORD PTR -24[rbp]
        imul    eax, DWORD PTR -20[rbp]
        imul    eax, DWORD PTR -16[rbp]
        imul    eax, DWORD PTR -12[rbp]
        imul    eax, DWORD PTR -8[rbp]
        mov     edx, eax
        mov     eax, DWORD PTR -4[rbp]
        add     eax, edx
        pop     rbp
        ret
        .size   foobar, .-foobar
        .globl  main
        .type   main, @function
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        push    99
        mov     r9d, 88
        mov     r8d, 77
        mov     ecx, 66
        mov     edx, 55
        mov     esi, 44
        mov     edi, 33
        call    foobar
        add     rsp, 8
        mov     DWORD PTR -4[rbp], eax
        mov     eax, DWORD PTR -4[rbp]
        add     eax, 1
        leave
        ret
        .size   main, .-main
        .ident  "GCC: (GNU) 8.2.1 20180831"
        .section        .note.GNU-stack,"",@progbits
```
caller负责把前面六个参数入寄存器（整型和指针类型，其他类型请参考规范calling conventions），依次是edi,esi,edx,ecx,r8d,r9d，第七个以后的参数入栈。说明一下，由于这里的参数都是整型，4字节，所以寄存器都是用的4字节寄存器，如果是8字节，e改为r，例如rdi

callee会先保存caller的rbp（当前的frame pointer），然后保存当前rsp（strack pointer）为自己栈帧的rbp，接下来移动rsp，分配栈空间给参数或者内部变量使用。

callee会把返回结果放入到eax，如果结果超过8byte，可以增加一个寄存器edx来存放返回结果，如果超过16字节，可能会更复杂点，caller会在自己的栈分配一块空间，然后把指针当成callee的第一个参数放入rdi，其他参数依次往后移一位。caller可以从eax获取到返回结果（或者更复杂）。可以看kernel代码`arch/x86/entry/calling.h`:

``` assembly
 x86 function call convention, 64-bit:
 -------------------------------------
  arguments           |  callee-saved      | extra caller-saved | return
 [callee-clobbered]   |                    | [callee-clobbered] |
 ---------------------------------------------------------------------------
 rdi rsi rdx rcx r8-9 | rbx rbp [*] r12-15 | r10-11             | rax, rdx [**]

 ( rsp is obviously invariant across normal function calls. (gcc can 'merge'
   functions when it sees tail-call optimization possibilities) rflags is
   clobbered. Leftover arguments are passed over the stack frame.)

 [*]  In the frame-pointers case rbp is fixed to the stack frame.

 [**] for struct return values wider than 64 bits the return convention is a
      bit more complex: up to 128 bits width we return small structures
      straight in rax, rdx. For structures larger than that (3 words or
      larger) the caller puts a pointer to an on-stack return struct
      [allocated in the caller's stack frame] into the first argument - i.e.
      into rdi. All other arguments shift up by one in this case.
      Fortunately this case is rare in the kernel.
```


如果编译的时候启用了`-fomit-frame-pointer`选项，会优化掉rbp的使用，也就是不使用rbp做frame pointer，优化后的结果：

``` assembly
main:
        sub     rsp, 16
        push    99
        mov     r9d, 88
        mov     r8d, 77
        mov     ecx, 66
        mov     edx, 55
        mov     esi, 44
        mov     edi, 33
        call    foobar
        add     rsp, 8
        mov     DWORD PTR 12[rsp], eax
        mov     eax, DWORD PTR 12[rsp]
        add     eax, 1
        add     rsp, 16
        ret
        .size   main, .-main
        .ident  "GCC: (GNU) 8.2.1 20180831"
        .section        .note.GNU-stack,"",@progbits
```

# x84-64 date types
| C declaration | Intel data type | GAS suffix | x86-64 Size (Bytes) |
| --- | --- | --- | --- |
| char | Byte |  b | 1 |
| short |  Word | w  | 2 |
| int | Double word | l | 4 |
| unsigned | Double word | l | 4 |
| long | int | Quad word | q | 8 |
| unsigned long | Quad word | q | 8 |
| char * | Quad word | q | 8 |
| float | Single precision | s | 4 |
| double | Double precision | d | 8 |
| long double | Extended precision | t | 16 |

char * 指针64bit

# PIE / PIC[^3][^4][^5]
在我的archlinux上（2018年12月的更新），gcc默认启用了pic和pie，会导致加载地址随机：

PIC全名Postion Independent Codes，实现共享库的加载地址不固定，地址需要loader在加载时确定。

PIE是指Position Independent Executables, 实现可执行文件的加载地址不固定（没有PIE之前，可执行文件默认会从固定地址加载，例如0x400000作为base加载地址[^6]）。
```
➜  cat simple.c
#include <stdio.h>
int foobar(int *a, int b, int c, int d, int e, int f, int g)
{
    int aa = *a + 1;
    int bb = b + 2;
    int cc = c + 3;
    int dd = d + 4;
    int ee = e + 5;
    int ff = f + 6;
    int gg = g + 6;
    int sum = aa + bb + cc + dd + ee + ff + gg;

    return aa * bb * cc * dd * ee * ff * gg + sum;
}

int main()
{
    int b = 111;
    int a = foobar(&b, 44, 55, 66, 77, 88, 99);
    printf("hello world");
    return a + 1;
}
```

## 默认编译（-fPIC -fPIE）：
```
➜  binary-study > objdump -t simple|grep text
0000000000001050 l    d  .text  0000000000000000              .text
0000000000001080 l     F .text  0000000000000000              deregister_tm_clones
00000000000010b0 l     F .text  0000000000000000              register_tm_clones
00000000000010f0 l     F .text  0000000000000000              __do_global_dtors_aux
0000000000001140 l     F .text  0000000000000000              frame_dummy
00000000000012e0 g     F .text  0000000000000005              __libc_csu_fini
0000000000001149 g     F .text  00000000000000a3              foobar
0000000000001270 g     F .text  0000000000000065              __libc_csu_init
0000000000001050 g     F .text  000000000000002f              _start
00000000000011ec g     F .text  000000000000007b              main

➜  binary-study > gdb simple
...
Reading symbols from simple...(no debugging symbols found)...done.
(gdb) b main
Breakpoint 1 at 0x11f0    # 这个地址不是虚拟地址，真正的地址需要运行时确认
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x00000000000011f0 <main+4>
(gdb) run
Starting program: /home/tacy/workspace/binary-study/simple

Breakpoint 1, 0x00005555555551f0 in main ()   # 地址确认，替换成真正的虚拟地址
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x00005555555551f0 <main+4>
        breakpoint already hit 1 time
(gdb) info address main
Symbol "main" is at 0x5555555551ec in a file compiled without debugging.


➜  binary-study >  cat /proc/`pidof simple`/maps   # 加载地址，不是传统的0x4000000
555555554000-555555555000 r--p 00000000 08:06 10403523                   /home/tacy/workspace/binary-study/simple
555555555000-555555556000 r-xp 00001000 08:06 10403523                   /home/tacy/workspace/binary-study/simple
555555556000-555555557000 r--p 00002000 08:06 10403523                   /home/tacy/workspace/binary-study/simple
555555557000-555555558000 r--p 00002000 08:06 10403523                   /home/tacy/workspace/binary-study/simple
555555558000-555555559000 rw-p 00003000 08:06 10403523                   /home/tacy/workspace/binary-study/simple
```
编译的时候开启pic和pie，symbol的地址需要在运行时确认（必须先运行才知道地址）

## 禁用PIE和PIC

```
➜  binary-study gcc -no-pie simple.c -o simple
➜  binary-study objdump -t simple|grep text
0000000000401050 l    d  .text  0000000000000000              .text
0000000000401090 l     F .text  0000000000000000              deregister_tm_clones
00000000004010c0 l     F .text  0000000000000000              register_tm_clones
0000000000401100 l     F .text  0000000000000000              __do_global_dtors_aux
0000000000401130 l     F .text  0000000000000000              frame_dummy
00000000004012d0 g     F .text  0000000000000005              __libc_csu_fini
0000000000401136 g     F .text  00000000000000a3              foobar
0000000000401260 g     F .text  0000000000000065              __libc_csu_init
0000000000401080 g     F .text  0000000000000005              .hidden _dl_relocate_static_pie
0000000000401050 g     F .text  000000000000002f              _start
00000000004011d9 g     F .text  0000000000000079              main
➜  binary-study gdb simple
...
Reading symbols from simple...(no debugging symbols found)...done.
(gdb) info address main
Symbol "main" is at 0x4011d9 in a file compiled without debugging.
(gdb) b main
Breakpoint 1 at 0x4011dd
(gdb) run
Starting program: /home/tacy/workspace/binary-study/simple

Breakpoint 1, 0x00000000004011dd in main ()
(gdb)

➜  binary-study cat /proc/`pidof simple`/maps
00400000-00401000 r--p 00000000 08:06 10358659                           /home/tacy/workspace/binary-study/simple
00401000-00402000 r-xp 00001000 08:06 10358659                           /home/tacy/workspace/binary-study/simple
00402000-00403000 r--p 00002000 08:06 10358659                           /home/tacy/workspace/binary-study/simple
00403000-00404000 r--p 00002000 08:06 10358659                           /home/tacy/workspace/binary-study/simple
00404000-00405000 rw-p 00003000 08:06 10358659                           /home/tacy/workspace/binary-study/simple

```

## 启用enable-default-pie编译选项的gcc

如果你的gcc启用了enable-default-pie，那么你编译程序的时候，默认就带上`-fPIC -fPIE`参数（可以通过-Q -v观察）


想要禁用pic/pie，按照下面方式编译：

``` shell
➜  binary-study> gcc -mcmodel=large -fno-PIC -c piepic.c -o piepic.o
➜  binary-study> gcc -mcmodel=large -no-pie -Wl,-O1,--sort-common,--as-needed,-z,relro,-z,now  -shared -o libpiepic.so piepic.o
➜  binary-study> file libpiepic.so
libpiepic.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=3b01764ad66d87aa17a43c5cc52ccae070638ad6, not stripped
➜  binary-study readelf -r libpiepic.so

Relocation section '.rela.dyn' at offset 0x3f8 contains 12 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
0000000010ef  000000000008 R_X86_64_RELATIVE                    400c
0000000010f9  000000000008 R_X86_64_RELATIVE                    2000
000000001114  000000000008 R_X86_64_RELATIVE                    2012
000000003e08  000000000008 R_X86_64_RELATIVE                    10e0
000000003e10  000000000008 R_X86_64_RELATIVE                    1090
000000004000  000000000008 R_X86_64_RELATIVE                    4000
000000001108  000300000001 R_X86_64_64       0000000000000000 printf@GLIBC_2.2.5 + 0
00000000111e  000200000001 R_X86_64_64       0000000000000000 puts@GLIBC_2.2.5 + 0
000000003fe0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000003fe8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000003ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0
➜  binary-study> gcc -no-pie -o client_nopie client_piepic.c libpiepic.so
➜  binary-study ./client_nopie   # ASLR设置为1的情况
shared_v at 0xca3f800c
hello
value at 0x40403c
➜  binary-study ./client_nopie   # ASLR设置为1的情况，设置为2，全部随机加载了
shared_v at 0xa421900c
hello
value at 0x40403c
```

默认情况下的编译方式（启用pie/pic）：

``` shell
➜  binary-study gcc -c piepic.c -o piepic.o
➜  binary-study gcc -Wl,-O1,--sort-common,--as-needed,-z,relro,-z,now  -shared -o libpiepic.so piepic.o
➜  binary-study readelf -r libpiepic.so

Relocation section '.rela.dyn' at offset 0x3f8 contains 7 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003dc8  000000000008 R_X86_64_RELATIVE                    1110
000000003dd0  000000000008 R_X86_64_RELATIVE                    10c0
000000004000  000000000008 R_X86_64_RELATIVE                    4000
000000003fe0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000003fe8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000003ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x4a0 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003fd0  000200000007 R_X86_64_JUMP_SLO 0000000000000000 puts@GLIBC_2.2.5 + 0
000000003fd8  000300000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
➜  binary-study gcc -o client client_piepic.c libpiepic.so
➜  binary-study ./client
shared_v at 0x3d4d900c
hello
value at 0xb29303c
➜  binary-study ./client
shared_v at 0xbbe9700c
hello
value at 0xc892303c
```

不启用pic和pie，symbol的地址在编译的时候就确认了，可以直接使用

0 > /proc/sys/kernel/randomize_va_space

# readelf
elf格式工具，可以用来查看elf文件信息，例如segment，header，relocs等


# gcc

## 编译共享库
gcc -march=x86-64 -mtune=generic -O2 -pipe -fstack-protector-strong -fno-plt -fno-PIC -c piepic.c -o piepic.o

显示编译选项启用情况： gcc -Q -v <input file>

有main：
gcc -fno-PIC  -no-pie -fno-stack-protector -c simple.c -o simple.o
gcc -shared -no-pie -o libsimple.so simple.o  # no-pie必须在shared后面

# resource
1. [Notes on x86-64 programming](/resource/x86-64.pdf)
2. [x86-64 Machine-Level Programming](/resource/asm64-handout.pdf)
3. [Introduction to the ELF Format](https://blog.k3170makan.com/2018/09/introduction-to-elf-format-elf-header.html)


[^1]: [stack frame for x86](https://www.cs.rutgers.edu/~pxk/419/notes/frames.html)

[^2]: [stack frame for x64](Stack frame layout on x86-64)

[^3]: [about ELF – PIE, PIC and else](https://codywu2010.wordpress.com/2014/11/29/about-elf-pie-pic-and-else/)

[^4]: [Load-time relocation of shared libraries](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries)

[^5]: [Position Independent Code (PIC) in shared libraries on x64](https://eli.thegreenplace.net/2011/11/11/position-independent-code-pic-in-shared-libraries-on-x64)

[^6]: [Why is 0x00400000 the default base address for an executable?](https://blogs.msdn.microsoft.com/oldnewthing/20141003-00/?p=43923)
