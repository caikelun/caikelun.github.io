# Android APP native 崩溃分析之令人困惑的 backtrace

* Author: Kelun Cai (caikelun@gmail.com)
* Date: 2019-06-01
* Tags: Android, crash
* License: [Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)


# 现象

近期，业务方反馈了一个奇怪的崩溃问题，认为信息不足，无法解决。

```
Signal: 11 (SIGSEGV), Code: 1 (SEGV_MAPERR), fault addr 0x1

r0  993ff520  r1  dc3170c4  r2  00000000  r3  dabe3e08
r4  993ff520  r5  00000005  r6  00000290  r7  000007ac
r8  e83253a0  r9  00006aba  r10 bf921e39  r11 e83253a0
ip  bfa3a9e0  sp  993ff494  lr  bf88a71d  pc  bf96c31c

#00 pc 001a731c  /data/data/com.package.name/files/download/libmcto_media_player.so
#01 pc 0020b7e5  /data/data/com.package.name/files/download/libmcto_media_player.so

#00  993ff494  0000022c
     993ff498  adcfd000  [anon:libc_malloc]
     993ff49c  bf88a71d  /data/data/com.package.name/files/download/libmcto_media_player.so
     993ff4a0  ffffffff
     993ff4a4  ffffffff
     993ff4a8  bf9d07e7  /data/data/com.package.name/files/download/libmcto_media_player.so
#01  993ff4ac  00000000
     993ff4b0  00000000
     993ff4b4  00000000
     993ff4b8  00000000
     993ff4bc  00000000
     993ff4c0  adcfd234  [anon:libc_malloc]
     993ff4c4  00000000
     993ff4c8  0000006e
     993ff4cc  00000000
     993ff4d0  adcfdf6c  [anon:libc_malloc]
     993ff4d4  00000000
     993ff4d8  00000000
     993ff4dc  00000000
     993ff4e0  00000000
     993ff4e4  00000000
     993ff4e8  00000000
```

第一感觉这肯定是动态库的业务逻辑有 bug，导致了段错误。backtrace 确实是不完整的，但一下也看不出为什么不完整，确实需要协助分析一下。


# 分析

## backtrace 不完整的常见原因

backtrace 不完整的情况时有发生，常见的原因有：

* 崩溃时 stack 内存被大量误写。如果崩溃点附近的逻辑正在处理的外部输入随机性很大，情况就更加糟糕，往往会看到大量离散的不完整 backtrace。举例：

```
#00 pc 00000ffb  <anonymous:c34fe000>
#01 pc 0009a885  /data/app/com.package.name-1/lib/arm/libjsc.so
#02 pc 0003ff93  /data/app/com.package.name-1/lib/arm/libjsc.so
#03 pc 0011f60f  /data/app/com.package.name-1/lib/arm/libjsc.so
#04 pc fffffffb  <unknown>
```

```
#00 pc 000092fe  <anonymous:c15d0000>
#01 pc 00099ec3  /data/app/com.package.name-1/lib/arm/libjsc.so
#02 pc 00003ffe  <anonymous:bf140000>
```

```
#00 pc 00000ffb  <anonymous:ef304000>
```

* 调用路径上的某些 ELF 文件的 unwind table 不完整。比如某些系统的 odex/oat，还有系统的 WebView Chromium，都属于此类。举例：

```
#00 pc 00d12bcc  /system/lib/libwebviewchromium.so
```

```
#00 pc 01a0cf72  /system/app/WebViewGoogle/WebViewGoogle.apk!libwebviewchromium.so (offset 0x46da000)
```

```
#00 pc 00006fde  /data/app/com.package.name-1/lib/arm/libcros.so
#01 pc 00007007  /data/app/com.package.name-1/lib/arm/libcros.so
#02 pc 00007023  /data/app/com.package.name-1/lib/arm/libcros.so
#03 pc 00007037  /data/app/com.package.name-1/lib/arm/libcros.so
#04 pc 000070d1  /data/app/com.package.name-1/lib/arm/libcros.so
#05 pc 000049bf  /data/app/com.package.name-1/lib/arm/libcros.so
#06 pc 000092e3  /data/app/com.package.name-1/oat/arm/base.odex
```

```
#00 pc 00013792  /system/lib/libc.so (__futex_wait_ex+49)
#01 pc 00013b21  /system/lib/libc.so (pthread_mutex_lock+310)
#02 pc 00028351  /system/lib/libc.so (dlfree+48)
#03 pc 0000ef33  /system/lib/libc.so (free+10)
#04 pc 0000a367  /system/lib/libjavacrypto.so
#05 pc 0000bc4d  /system/lib/libjavacrypto.so
#06 pc 022fd081  /system/framework/arm/boot.oat
```

* 调用路径上的某些 ELF 文件本身损坏了，或被移除了。另外，如果崩溃点本身位于损坏的 ELF 中，有时收到的信号会是 SIGBUS。举例：

```
#00 pc 5d9840f2  <unknown>
#01 pc 4008ab6c  <unknown>
```

```
#00 pc 00392fd0  /system/lib/egl/libGLES_mali.so
#01 pc 0002ab7b  /system/lib/libgui.so (_ZN7android10GLConsumer22bindTextureImageLockedEv+182)
#02 pc 0002b3a9  /system/lib/libgui.so (_ZN7android10GLConsumer14updateTexImageEv+208)
#03 pc b3317c6c  <unknown>
```

* 执行的指令位于 SharedMemory 中，此时读取到的 ELF 内容可能是不可靠的，为了避免误导，一般都会选择主动终止 unwind。举例：

```
#00 pc 0007a010 /dev/ashmem/dalvik-jit-code-cache (deleted)
```

```
#00 pc 00019e64  /system/lib/libssl.so (SSL_clear+19)
#01 pc 000103b5  /system/lib/libjavacrypto.so (_ZL25NativeCrypto_SSL_shutdownP7_JNIEnvP7_jclassxP8_jobjectS4_+156)
#02 pc 00027a7d  /system/framework/arm/boot-conscrypt.oat (com.android.org.conscrypt.NativeCrypto.SSL_shutdown+156)
#03 pc 00032a03  /system/framework/arm/boot-conscrypt.oat (com.android.org.conscrypt.OpenSSLSocketImpl.shutdownAndFreeSslNative+138)
#04 pc 0003330b  /system/framework/arm/boot-conscrypt.oat (com.android.org.conscrypt.OpenSSLSocketImpl.close+434)
#05 pc 003e0931  /system/lib/libart.so (art_quick_invoke_stub_internal+64)
#06 pc 003e4ea3  /system/lib/libart.so (art_quick_invoke_stub+226)
#07 pc 000ac2d9  /system/lib/libart.so (_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc+140)
#08 pc 001f27fb  /system/lib/libart.so (_ZN3art11interpreter34ArtInterpreterToCompiledCodeBridgeEPNS_6ThreadEPNS_9ArtMethodEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+238)
#09 pc 001edd71  /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+576)
#10 pc 003cce3d  /system/lib/libart.so (MterpInvokeVirtualQuick+504)
#11 pc 003d6994  /system/lib/libart.so (ExecuteMterpImpl+29972)
#12 pc 001d5351  /system/lib/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+340)
#13 pc 001da6a3  /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142)
#14 pc 001edd5b  /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+554)
#15 pc 003cb927  /system/lib/libart.so (MterpInvokeStatic+322)
#16 pc 003d2d94  /system/lib/libart.so (ExecuteMterpImpl+14612)
#17 pc 001d5351  /system/lib/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+340)
#18 pc 001da6a3  /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142)
#19 pc 001ee931  /system/lib/libart.so (_ZN3art11interpreter6DoCallILb1ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+420)
#20 pc 003cc9eb  /system/lib/libart.so (MterpInvokeDirectRange+294)
#21 pc 003d3014  /system/lib/libart.so (ExecuteMterpImpl+15252)
#22 pc 001d5351  /system/lib/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+340)
#23 pc 001da6a3  /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142)
#24 pc 001ee931  /system/lib/libart.so (_ZN3art11interpreter6DoCallILb1ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+420)
#25 pc 003cc9eb  /system/lib/libart.so (MterpInvokeDirectRange+294)
#26 pc 003d3014  /system/lib/libart.so (ExecuteMterpImpl+15252)
#27 pc 001d5351  /system/lib/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+340)
#28 pc 001da6a3  /system/lib/libart.so (_ZN3art11interpreter33ArtInterpreterToInterpreterBridgeEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameEPNS_6JValueE+142)
#29 pc 001edd5b  /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+554)
#30 pc 003cce3d  /system/lib/libart.so (MterpInvokeVirtualQuick+504)
#31 pc 003d6994  /system/lib/libart.so (ExecuteMterpImpl+29972)
#32 pc 001d5351  /system/lib/libart.so (_ZN3art11interpreterL7ExecuteEPNS_6ThreadEPKNS_7DexFile8CodeItemERNS_11ShadowFrameENS_6JValueEb+340)
#33 pc 001da5f1  /system/lib/libart.so (_ZN3art11interpreter30EnterInterpreterFromEntryPointEPNS_6ThreadEPKNS_7DexFile8CodeItemEPNS_11ShadowFrameE+92)
#34 pc 003c0fbd  /system/lib/libart.so (artQuickToInterpreterBridge+944)
#35 pc 003e46f1  /system/lib/libart.so (art_quick_to_interpreter_bridge+32)
#36 pc 000a5511  /dev/ashmem/dalvik-jit-code-cache (deleted)
```

## 崩溃位置的初步分析

回到问题本身。先看一下崩溃位置 `001a731c`：

```
.text:001A7310  STMFD   SP!, {R4,R5,LR}
.text:001A7314  LDR     R5, [R1]
.text:001A7318  MOV     R4, R0
.text:001A731C  LDR     R3, [R5,#-4] ;崩溃发生在这里
.text:001A7320  SUB     SP, SP, #0xC
.text:001A7324  CMP     R3, #0
.text:001A7328  SUB     R0, R5, #0xC
.text:001A732C  BLT     loc_1A7350
.text:001A7330  LDR     R3, =(dword_2759D4 - 0x1A733C)
.text:001A7334  ADD     R3, PC, R3 ; dword_2759D4
.text:001A7338  CMP     R0, R3
.text:001A733C  BNE     loc_1A7364
.text:001A7340 loc_1A7340
.text:001A7340  STR     R5, [R4]
.text:001A7344  MOV     R0, R4
.text:001A7348  ADD     SP, SP, #0xC
.text:001A734C  LDMFD   SP!, {R4,R5,PC}
.text:001A7350  ADD     R1, SP, #0x18+var_14
.text:001A7354  MOV     R2, #0
.text:001A7358  BL      sub_1A6EA8
.text:001A735C  MOV     R5, R0
.text:001A7360  B       loc_1A7340
.text:001A7364  MOV     R1, #1
.text:001A7368  ADD     R0, R0, #8
.text:001A736C  BL      sub_1C2CAC
.text:001A7370  B       loc_1A7340
```

这是一个相对比较短的完整的调用过程。先压栈了 `R4`、`R5`、`LR`，然后开始执行。执行到 `LDR R3, [R5,#-4]` 时发生了段错误，`R5` 的当前值是 `0x5`，几乎不用查 maps 也能知道 `0x1` (`0x5 - 0x4 = 0x1`) 这个虚拟内存地址肯定是非法的，发生段错误是正常的。Signal code `SEGV_MAPERR` 和 fault addr `0x1` 也完全符合预期。

由于只有两行 backtrace，我们继续看下一行，在 `0020b7e5`:

```
............
.rodata:0020B795              DCB "try_count_=%d",0
.rodata:0020B7E3 asc_20B7E3   DCB "://",0 
.rodata:0020B7E7 aCdn         DCB "CDN",0
............
```

出乎意料的是 `0020b7e5` 位于 `.rodata` 中，但这也正好解释了为什么 unwind 过程中断了（backtrace 不完整）。

## 可疑之处

再次回看崩溃位置附近的指令，确实发现了可疑之处：

```
.text:001A7310  STMFD   SP!, {R4,R5,LR}
............
.text:001A731C  LDR     R3, [R5,#-4] ;崩溃发生在这里
.text:001A7320  SUB     SP, SP, #0xC
............
.text:001A7348  ADD     SP, SP, #0xC
.text:001A734C  LDMFD   SP!, {R4,R5,PC}
```

在这个相对较短的调用过程中，一共才使用了 24 字节的栈内存空间，但是 `SP` 居然并没有一次性移动到位，这是很不寻常的。

## unwind table

看一下 unwind table：

```
$ arm-linux-androideabi-readelf -u ./libmcto_media_player.so

............

0x1a7268: 0x80b108ab
  Compact model index: 0
  0xb1 0x08 pop {r3}
  0xab      pop {r4, r5, r6, r7, r14}
  
0x1a7310: 0x8002a9b0
  Compact model index: 0
  0x02      vsp = vsp + 12
  0xa9      pop {r4, r5, r14}
  0xb0      finish
  
0x1a7424: 0x8001a8b0
  Compact model index: 0
  0x01      vsp = vsp + 8
  0xa8      pop {r4, r14}
  0xb0      finish
............
```

崩溃位置 `001a731c` 匹配了起始 offset 为 `1a7310` 的 unwind 信息码 `0x8002a9b0`，根据这里的信息，unwind 时 `SP` 的值一共需要增加 24 个字节（出栈 24 个字节的数据）。但是从前述的汇编指令可以看出，当崩溃发生时（执行到 `001a731c`），`SP` 的值只减少了 12 个字节（`STMFD SP!, {R4,R5,LR}`），这就是问题所在了。

## 再看 stack

根据 stack 中的数据：

```
#00  993ff494  0000022c
     993ff498  adcfd000  [anon:libc_malloc]
     993ff49c  bf88a71d  /data/data/com.package.name/files/download/libmcto_media_player.so
     993ff4a0  ffffffff
     993ff4a4  ffffffff
     993ff4a8  bf9d07e7  /data/data/com.package.name/files/download/libmcto_media_player.so
#01  993ff4ac  00000000
     993ff4b0  00000000
     993ff4b4  00000000
............
```

我们看到 unwind 过程其实是严格按照 unwind table 中的信息进行的，于是就被误导了，`SP` 比实际需要的多移动了 12 个字节，真正的 `LR` 保存在内存地址 `993ff49c` 中，它的值是 `bf88a71d`，根据 maps 信息，我们计算出了这个绝对地址相对于当前 ELF 的 offset，可惜由于业务方动态库的逻辑比较复杂，调用层次也比较深，而 unwind 过程过早的终止了，仅凭现有的寄存器、stack 和 memory 等信息，已经不足以帮助业务方定位问题。

## 001a731c 处究竟是什么？

这种 “unwind table 信息” 与 “对应的汇编指令序列” 相矛盾的情况并不常见。`001a731c` 处究竟是什么函数？为什么会有这样的指令序列 ？

从业务方那里拿到了带调试符号的动态库文件：

```
$arm-linux-androideabi-addr2line -f -e ./libmcto_media_player.so 001a731c
_ZNSsC1ERKSs
libgcc2.c:?

$arm-linux-androideabi-c++filt -n _ZNSsC1ERKSs
std::basic_string<char, std::char_traits<char>, std::allocator<char> >::basic_string(std::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)
```

原来是 `std::basic_string` 的构造函数。

这个问题很有可能是 NDK 的 bug 造成的。从业务方那里了解到，他们用的 NDK 版本是 r9d。

## 了解你使用的 NDK

开发和维护一套跨平台的交叉编译工具绝非易事。相对于 C 语言，编译器需要付出很多额外的努力，才能确保 C++ 的各种语法特性在运行时能够如预期的那样工作，C++ 标准库长久以来也是各种不同版本共存的局面。要支持 Android 新版本对底层的改动，又要维护向后的兼容性。NDK 其实并不如我们预想的那样稳定可靠。可以到 NDK 的 github 官方 [issues](https://github.com/android-ndk/ndk/issues) 了解一下现状。

NDK 从 r11 开始才在 Changelog 里明确的列出了重要的 Known Issues。

我们看到在 [r11 Changelog](https://android.googlesource.com/platform/ndk/+/refs/heads/ndk-r11-release/CHANGELOG.md) 的 Known Issues 中赫然写到：

> Exception handling will often fail when using c++_shared on ARM32. The root cause is incompatibility between the LLVM unwinder used by libc++abi for ARM32 and libgcc. This is not a regression from r10e.

在 [r12 Changelog](https://android.googlesource.com/platform/ndk/+/refs/heads/ndk-r11-release/CHANGELOG.md) 的 Known Issues 中写到：

> Exception unwinding with c++_shared still does not work for ARM on Gingerbread or Ice Cream Sandwich.

我们知道 C++ 的异常处理机制在运行时也是依赖于 unwind 的。问题应该就出在这里了。


# 结论

业务方使用较新版本的 NDK 重新编译了动态库，我们检查了 `std::basic_string` 对应的汇编指令，发现这次 `SP` 在函数开头处一次性移动到位了。应该没有问题了。业务方上线重新编译的动态库，拿到完整的 backtrace 后，就可以定位并修复这个段错误问题了。

因此，这次 backtrace 不完整问题的原因是：**低版本 NDK 的 bug，导致生成的动态库在某些情况下无法正确的执行 unwind**。

根据前述 Known Issues 中的描述，不但崩溃后的 backtrace 获取有时会受到影响，业务逻辑自身用到 C++ 异常机制的地方，也有可能会受到影响，具体来说就是：抛出异常后，可能无法如代码逻辑预期的那样去逐层的执行异常捕获逻辑。如果真的存在这样的隐藏问题，希望这次的 NDK 版本升级也能把它们一起修复了。


# 关于崩溃捕获工具

最后又到了我们的广告时间。

上述所有的线上崩溃信息，都是使用我们自己开发的 Android APP 崩溃捕获工具 [xCrash](https://github.com/iqiyi/xCrash) 捕获的。
