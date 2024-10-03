# Android APP native 崩溃分析之时间的秘密

* Author: Kelun Cai (caikelun@gmail.com)
* Date: 2019-08-01
* Tags: Android, crash
* License: [Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)


# 现象

## 现象 1

```
Signal: 7 (SIGBUS), Code: 2 (BUS_ADRERR), fault addr 0xcd596372

r0  0000006e  r1  00000000  r2  00000000  r3  00000000
r4  f35bb740  r5  f35bb748  r6  cd3ff8a8  r7  00000064
r8  f7451e34  r9  f35bb708  r10 cd301000  r11 cd5a3ac5
ip  00000000  sp  cd3ff890  lr  00000000  pc  cd596372

#00 pc 0009a372  /data/data/com.package.name/files/download/libCube.so
#01 pc 000a4f69  /data/data/com.package.name/files/download/libCube.so
#02 pc 000a513f  /data/data/com.package.name/files/download/libCube.so
#03 pc 000a7ab9  /data/data/com.package.name/files/download/libCube.so
#04 pc 000a7abf  /data/data/com.package.name/files/download/libCube.so
#05 pc 000a7add  /data/data/com.package.name/files/download/libCube.so
#06 pc 0004078b  /system/lib/libc.so (_ZL15__pthread_startPv+30)
#07 pc 0001a031  /system/lib/libc.so (__start_thread+6)

#00  cd3ff890  cd3ff8dc
     cd3ff894  5d419f43
     cd3ff898  cd3ff8a0
     cd3ff89c  f35bb744  [anon:libc_malloc]
     cd3ff8a0  5d419f43
     cd3ff8a4  000a5952
     cd3ff8a8  5d419f43
     cd3ff8ac  2e62c950  /dev/ashmem/dalvik-main space (deleted)
     cd3ff8b0  00000064
     cd3ff8b4  f35bb700  [anon:libc_malloc]
     cd3ff8b8  f35bb750  [anon:libc_malloc]
     cd3ff8bc  00000064
     cd3ff8c0  cd3ff8dc
     cd3ff8c4  cd5a0f6d  /data/data/com.package.name/files/download/libCube.so
#01  cd3ff8c8  cd5a3ac5  /data/data/com.package.name/files/download/libCube.so
     cd3ff8cc  00000000
     cd3ff8d0  00000001
     cd3ff8d4  262237a1  /dev/ashmem/dalvik-main space (deleted)
     cd3ff8d8  00000000
......

......
memory near pc:
    cd596350 43591889 60714b11 dd054299 33019b01  ..YC.Kq`.B.....3
    cd596360 4b0f9306 607118c9 1c299803 f0251c32  ...K..q`..).2.%.
    cd596370 286ef8d5 7b23d003 d0f52b00 2601e002  ..n(..#{.+.....&
    cd596380 e0004276 78232600 d1002b00 1c287323  vB...&#x.+..#s(.
    cd596390 fd84f024 b0091c30 46c0bdf0 3b9ac9ff  $...0......F...;
    cd5963a0 c4653600 1c0cb510 f8c0f025 d1022800  .6e.....%....(..
    cd5963b0 20016020 f024e004 6800fcd1 20006020   `. ..$....h `. 
    cd5963c0 b570bd10 26006803 3b0cb09a 1c0c681b  ..p..h.&...;.h..
    cd5963d0 d10342b3 600a2202 e0161c18 6800600e  .B...".`.....`.h
    cd5963e0 f0244669 2800fef3 f024d005 6800fcb7  iF$....(..$....h
    cd5963f0 1c306020 9b04e009 021222f0 2380401a   `0......"...@.#
    cd596400 429a021b 6020d101 b01a2001 b538bd70  ...B.. `. ..p.8.
    cd596410 1c0d6800 3b0c1c03 2c00681c 2302d102  .h.....;.h.,...#
    cd596420 e00d600b feeaf024 d0051e04 602b2300  .`..$........#+`
    cd596430 fef4f024 e0042001 fc90f024 60286800  $.... ..$....h(`
    cd596440 bd381c20 6803b570 1c0d1c06 681c3b0c   .8.p..h.....;.h
......

......
cd4fc000-cd5e9000 r-x       0  ed000 /data/data/com.package.name/files/download/libCube.so
cd5e9000-cd5ee000 r--   ec000   5000 /data/data/com.package.name/files/download/libCube.so
cd5ee000-cd5ef000 rw-   f1000   1000 /data/data/com.package.name/files/download/libCube.so
......

```

```
.text:0009A368      LDR     R0, [SP,#0x38+var_2C]
.text:0009A36A      MOVS    R1, R5
.text:0009A36C      MOVS    R2, R6
.text:0009A36E      BL      j_pthread_cond_timedwait
.text:0009A372      CMP     R0, #0x6E ;在这个位置发生了 SIGBUS
.text:0009A374      BEQ     loc_9A37E
```

崩溃位置在 libCube.so 的 offset `0009a372` 处，根据 arm32 fastcall 调用约定，这里在 `pthread_cond_timedwait()` 函数返回后，检查 `r0` 中的返回值是否为 `6e` （十进制：`110`），查 `pthread_cond_timedwait()` manpage 和 Linux errno list 得知：`pthread_cond_timedwait()` 返回 110（ETIMEDOUT） 表示超时。

这是一个正常的调用，没有问题。那为什么会 SIGBUS 呢？难道是偶发的 NAND Flash 硬件故障？或者是驱动的 bug ？

从 [xCrash](https://github.com/iqiyi/xCrash) 捕获的崩溃信息中可以看到， 当前 pc 位置（`0009a372`）的二进制指令码是 `286e`，刚好对应 `CMP R0, #0x6E`，这里的指令码数据是通过 `ptrace()` 获取的，xCrash 能获取到但是 CPU 执行这条指令时却获取不到？


## 现象 2

```
Signal: 11 (SIGSEGV), Code: 1 (SEGV_MAPERR), fault addr 0x11d868

r0  bf206648  r1  00000000  r2  00000000  r3  00000000
r4  bf206640  r5  bf206648  r6  ffffffff  r7  000001b9
r8  d44bf980  r9  bf206608  r10 b1654000  r11 b21bd89d
ip  b240ed4c  sp  b1752890  lr  b21a66b9  pc  0011d868

#00 pc 0011d868  <unknown>
#01 pc 005586b5  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN3psl5Event4WaitEi+160)
#02 pc 0056fe55  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN8BaseHcdn6Thread9ThreadRunEv+122)
#03 pc 0057002b  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN8BaseHcdn10CThreadObj14ThreadWorkFuncEv+6)
#04 pc 0056f891  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN3psl13CThreadObject14ThreadBaseFuncEv+26)
#05 pc 0056f897  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN3psl13CThreadObject3RunEv+2)
#06 pc 0056f8b5  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN3psl10ThreadBaseINS_13CThreadObjectEE10ThreadProcEPv+24)
#07 pc 00040aab  /system/lib/libc.so (_ZL15__pthread_startPv+30)
#08 pc 0001a011  /system/lib/libc.so (__start_thread+6)

#00  b1752890  b17528dc
     ........  ........
#01  b1752890  b17528dc
     b1752894  5d419f83
     b1752898  b17528a0
     b175289c  bf206644  [anon:libc_malloc]
     b17528a0  5d419f83
     b17528a4  000cf94e
     b17528a8  5d419f84
     b17528ac  115c2ef0
     b17528b0  000001b9
     b17528b4  bf206600  [anon:libc_malloc]
     b17528b8  bf206650  [anon:libc_malloc]
     b17528bc  000001b9
     b17528c0  b17528dc
     b17528c4  b21bde59  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN8BaseHcdn6Thread9ThreadRunEv+126)
#02  b17528c8  b21bd89d  /data/data/com.package.name/files/download/libHCDNClientNet.so (_ZN3psl10ThreadBaseINS_13CThreadObjectEE10ThreadProcEPv)
     b17528cc  00000000
     b17528d0  00000001
     b17528d4  0730c6a9
     b17528d8  00000000
     b17528dc  bf20663c  [anon:libc_malloc]
     b17528e0  00000000
     b17528e4  bf206600  [anon:libc_malloc]
     b17528e8  b1752970
     b17528ec  b1752930
......

......
f73d0000-f7444000 r-x        0   74000 /system/lib/libc.so
f7444000-f7448000 r--    73000    4000 /system/lib/libc.so
f7448000-f744b000 rw-    77000    3000 /system/lib/libc.so
......
b1c4e000-b23ef000 r-x        0  7a1000 /data/data/com.package.name/files/download/libHCDNClientNet.so
b23ef000-b23f0000 ---        0    1000
b23f0000-b240f000 r--   7a1000   1f000 /data/data/com.package.name/files/download/libHCDNClientNet.so
b240f000-b241a000 rw-   7c0000    b000 /data/data/com.package.name/files/download/libHCDNClientNet.so
......
```

这是一个段错误，pc 的值是 `0011d868`，这个地址显然是异常的，我们俗称 pc “跑飞了”。backtrace 第二行中，libHCDNClientNet.so 的 offset `005586b5` 对应的指令如下：

```
.text:005586B2      MOVS    R0, R5
.text:005586B4      BL      j_pthread_mutex_unlock ;这里
.text:005586B8      MOVS    R0, R6
```

这是对 libc.so 中 `pthread_mutex_unlock()` 函数的调用，这里不应该有问题。难道是 linker 的 bug？`pthread_mutex_unlock()` 的绝对地址应该由 linker 写入 libHCDNClientNet.so 的 GOT 中。

此类怪异的崩溃现象还有一些，这里不继续罗列了。

问题到底出在哪里？

# 分析

## 问题特征

经过总结，发现上述的问题有以下几个特征：

* 从 tombstone 看不出问题的根本原因。
* 崩溃点本身都不在业务逻辑可控的范围内。
* 大多数发生在会导致线程 block 的调用的后一条指令处。
* 都发生在动态下载的 so 库中。（注意 so 库都在 `files` 目录中）
* 设备机型和操作系统版本分布无明显特征。
* 99% 以上发生在非 root 的设备上。（现在 root 手机的用户越来越少了）

## 崩溃捕获的机制

无论是安卓系统的 debuggerd，还是 [xCrash](https://github.com/iqiyi/xCrash)，或者 [Breakpad](https://chromium.googlesource.com/breakpad/breakpad/)，在捕获 native 崩溃时，都会经历以下几个阶段：

* 崩溃进程的 signal handler 被唤起执行。
* `clone()` 子进程。
* 在子进程中 `ptrace()` attach 到崩溃进程的各个线程。
* 读取崩溃进程中各个线程的寄存器、内存、ELF 等信息。
* 将信息直接写入 dump 文件；或者进一步分析提取 backtrace 等信息写入 dump 文件。

在子进程 attach 到所有崩溃进程的线程之前，崩溃进程仍然还在执行。如果考虑安卓 app 的多进程情况，那么在整个崩溃捕获期间，app 的其他进程也都可能在执行中。

崩溃捕获的机制也是由代码实现的，它们需要被一步一步的执行。我们可以想象：当案件（崩溃）发生时，警察（崩溃捕获机制）赶到案发现场是需要一定的时间的，这段时间就是 “视觉的盲区”。

对于我们前面描述的崩溃问题，在进程崩溃之后，但在 xCrash 开始执行并记录崩溃信息之前，一定还发生了一些我们所不知道的事情，案发现场的某些细节被篡改了！

## 可疑之处

回想上述的 “现象1”，指令码 `286e` 为什么 xCrash 能获取到，但是 CPU 执行时却获取不到而发生了 SIGBUS？

也许原因很简单：CPU 的执行流水线从内存加载这条指令时，so 库文件的这个 offset 位置确实不可读，于是发生了 SIGBUS；之后 xCrash 开始捕获崩溃信息，当 xCrash 用 `ptrace()` 从同一个内存位置读取数据时，so 库文件的这个 offset 位置又重新变的可读了。

最大的可能性是：so 库文件在已经被 linker 加载并开始执行的情况下，又再次被覆写了。也许是 so 库在动态下载、解压、校验的某些环节存在 bug，导致内部状态发生了异常。

## 寻找间接的证据

经过仔细的分析，我们确实发现了一些间接的证据：

### so 文件的最后修改时间

我们发了大量的 “崩溃 so 库文件的最后修改时间” 晚于 “崩溃时间点” 的情况。类似下面的情况：

```
process start time  : 2019-08-01 01:54:51.519 +0800
process crash time  : 2019-08-01 01:54:53.743 +0800
so last modify time : 2019-08-01 01:54:53.938 +0800
```

### so 文件的大小和 MD5 hash

由于动态下载的 so 库我们是可以从服务端可靠的获取的，我们比对了 xCrash 记录的崩溃 so 库的文件大小和 MD5 hash。发现有一部分确实是不对的，这部分情况应该就属于：“直到 xCrash 开始记录崩溃信息的时候，so 库被错误覆盖的过程还没有完全结束”。

### open files

通过分析 open files 列表，我们看到在有些情况下，崩溃进程仍然持有有效的 FD 可用于访问动态下载的 so 库所属的 zip 包：

```
Open files:
......
fd 57: /data/data/com.package.name/files/download/fullso.zip
......
```

有些 FD 还指向 zip 包的 crc 文件：

```
Open files:
......
fd 58: /data/data/com.package.name/files/download/fullso.zip.crc
......
```

# 结论

结论已经比较明显了：so 库在动态下载、解压、校验的某些环节存在 bug，导致 so 库文件在已经被 linker 加载并开始执行的情况下，又再次被覆写了。

后续：我们的 so 库动态下发流程做了改进，增加了更多防御性的设计。我们从 APM 线上崩溃数据的统计中看到，类似前述 “现象1” 和 “现象2” 的崩溃几乎都没有了，APP 的整体 native 崩溃率也进一步的降低了。
