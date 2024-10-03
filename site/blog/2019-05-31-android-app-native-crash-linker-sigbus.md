# Android APP native 崩溃分析之 linker SIGBUS 崩溃

* Author: Kelun Cai (caikelun@gmail.com)
* Date: 2019-05-31
* Tags: Android, crash
* License: [Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)


# 现象

## 现象 1

```
Signal: 7 (SIGBUS), Code: 2 (BUS_ADRERR)

r0  799963d8  r1  00000000  r2  00000be8  r3  3d800000
r4  6e1d5094  r5  00000003  r6  bebe53a4  r7  79998000
r8  ffffffff  r9  00000000  r10 799963d8  r11 00000000
ip  2670c8d5  sp  bebe5364  lr  26707915  pc  26708d54

#00 pc 00004d54  /system/bin/linker
#01 pc 00003911  /system/bin/linker
#02 pc 00003be5  /system/bin/linker
#03 pc 000023c1  /system/bin/linker
#04 pc 000029eb  /system/bin/linker
#05 pc 00000f43  /system/bin/linker
#06 pc 00052d97  /system/lib/libdvm.so (_Z17dvmLoadNativeCodePKcP6ObjectPPc+182)
#07 pc 0006a625  /system/lib/libdvm.so
#08 pc 000297e0  /system/lib/libdvm.so
#09 pc 00030c6c  /system/lib/libdvm.so (_Z11dvmMterpStdP6Thread+76)
#10 pc 0002e304  /system/lib/libdvm.so (_Z12dvmInterpretP6ThreadPK6MethodP6JValue+184)
#11 pc 00063719  /system/lib/libdvm.so (_Z15dvmInvokeMethodP6ObjectPK6MethodP11ArrayObjectS5_P11ClassObjectb+392)
#12 pc 0006b61f  /system/lib/libdvm.so
#13 pc 000297e0  /system/lib/libdvm.so
#14 pc 00030c6c  /system/lib/libdvm.so (_Z11dvmMterpStdP6Thread+76)
#15 pc 0002e304  /system/lib/libdvm.so (_Z12dvmInterpretP6ThreadPK6MethodP6JValue+184)
#16 pc 00063435  /system/lib/libdvm.so (_Z14dvmCallMethodVP6ThreadPK6MethodP6ObjectbP6JValueSt9__va_list+336)
#17 pc 0004cbb7  /system/lib/libdvm.so
#18 pc 0004dc37  /system/lib/libandroid_runtime.so
#19 pc 0004e95b  /system/lib/libandroid_runtime.so (_ZN7android14AndroidRuntime5startEPKcS2_+354)
#20 pc 0000105b  /system/bin/app_process
#21 pc 0000e49b  /system/lib/libc.so (__libc_init+50)
#22 pc 00000d7c  /system/bin/app_process
at java.lang.Runtime.nativeLoad(Native Method)
at java.lang.Runtime.doLoad(Runtime.java:435)
at java.lang.Runtime.load(Runtime.java:336)
at java.lang.System.load(System.java:533)
............

#00  bebe5364  799963d8  /data/data/com.package.name/files/download/libmctocurl.so
#01  bebe5368  0000004f
     bebe536c  00145000
     bebe5370  bebe53a4
     bebe5374  70ccfa00
     bebe5378  29648240  /dev/ashmem/dalvik-heap (deleted)
     bebe537c  27b9c8f0
     bebe5380  00000000
     bebe5384  00000001
     bebe5388  27c5cc38  /system/lib/libdvm.so (dvmCompilerTemplateStart+380)
     bebe538c  26707be9  /system/bin/linker
#02  bebe5390  00000000
     bebe5394  267063c5  /system/bin/linker
#03  bebe5398  27c62e18  /system/lib/libdvm.so
     bebe539c  69a75b1c
     bebe53a0  0000000c
     bebe53a4  70ccfa00
............
```

基本都出现在 Android 4.x 以及更低版本的设备上，dvm 调用 linker 加载动态库时发生了崩溃。由于 Android 早期的系统库为了压缩体积 strip 掉了大量的符号信息，所以仅从 backtrace 能获取到的信息偏少。

## 现象 2

```
Signal: 7 (SIGBUS), Code: 2 (BUS_ADRERR)

r0  00000000  r1  00000000  r2  000008d8  r3  a4ec86e8
r4  87e16094  r5  00000003  r6  a47034ec  r7  a4ed6000
r8  ffffffff  r9  00000000  r10 a4ec86e8  r11 00000000
ip  00000000  sp  a4703484  lr  400040fb  pc  40004870

#00 pc 00004870  /system/bin/linker (__dl_memset+48)
#01 pc 000040f7  /system/bin/linker (__dl__ZN9ElfReader12LoadSegmentsEv+238)
#02 pc 00004569  /system/bin/linker (__dl__ZN9ElfReader4LoadEPK17android_dlextinfo+40)
#03 pc 000023b3  /system/bin/linker (__dl__ZL12find_libraryPKciPK17android_dlextinfo+574)
#04 pc 00002543  /system/bin/linker (__dl__Z9do_dlopenPKciPK17android_dlextinfo+122)
#05 pc 00000e99  /system/bin/linker (__dl__ZL10dlopen_extPKciPK17android_dlextinfo+24)
#06 pc 001d4697  /system/lib/libart.so (_ZN3art9JavaVMExt17LoadNativeLibraryERKNSt3__112basic_stringIcNS1_11char_traitsIcEENS1_9allocatorIcEEEENS_6HandleINS_6mirror11ClassLoaderEEEPS7_+498)
#07 pc 001fa4a9  /system/lib/libart.so (_ZN3artL18Runtime_nativeLoadEP7_JNIEnvP7_jclassP8_jstringP8_jobjectS5_+480)
#08 pc 02324cc1  /system/framework/arm/boot.oat
at java.lang.Runtime.nativeLoad(Native Method)
at java.lang.Runtime.doLoad(Runtime.java:429)
at java.lang.Runtime.load(Runtime.java:330)
at java.lang.System.load(System.java:982)
............

#00  a4703484  a4ec86e8  /data/data/com.package.name/files/download/libcupid.so
#01  a4703488  000000e5
     a470348c  0079a000
     a4703490  00000000
     a4703494  a47034ec
     a4703498  00000000
     a470349c  00000000
     a47034a0  9706b430
     a47034a4  00000000
     a47034a8  a47034ec
     a47034ac  a4703548
     a47034b0  00000001
     a47034b4  4000456d  /system/bin/linker (__dl__ZN9ElfReader4LoadEPK17android_dlextinfo+44)
#02  a47034b8  00000000
     a47034bc  00000000
     a47034c0  0000b314
     a47034c4  400023b7  /system/bin/linker (__dl__ZL12find_libraryPKciPK17android_dlextinfo+578)
#03  a47034c8  427f9fec
     a47034cc  4246ca58
     a47034d0  4245c180
     a47034d4  9f45a800
............
```

基本都出现在 Android 5.x 的设备上，art 调用 linker 加载动态库时发生了崩溃。随着 Android 设备硬件配置的升级，对于系统库的符号多占用几兆 flash 空间已经不那么敏感，我们也能更加直观的从 backtrace 中看到 linker 中具体的崩溃位置。

## 现象 3

```
Signal: 7 (SIGBUS), Code: 2 (BUS_ADRERR)

r0  00000000  r1  00000000  r2  000008d8  r3  94ca06e8
r4  a5c67094  r5  00000003  r6  be837874  r7  94cae000
r8  ffffffff  r9  00000000  r10 94ca06e8  r11 00000001
ip  00000000  sp  be837814  lr  b6f8a069  pc  b6f8a7e0

#00 pc 000047e0  /system/bin/linker (__dl_memset+48)
#01 pc 00004065  /system/bin/linker (__dl__ZN9ElfReader12LoadSegmentsEv+232)
#02 pc 000044d9  /system/bin/linker (__dl__ZN9ElfReader4LoadEPK17android_dlextinfo+40)
#03 pc 000022f3  /system/bin/linker (__dl__ZL12find_libraryPKciPK17android_dlextinfo+578)
#04 pc 00002489  /system/bin/linker (__dl__Z9do_dlopenPKciPK17android_dlextinfo+128)
#05 pc 00000eb5  /system/bin/linker (__dl__ZL10dlopen_extPKciPK17android_dlextinfo+24)
#06 pc 00020d5d  /data/data/com.package.name/files/download/libCube.so
#07 pc 00020599  /data/data/com.package.name/files/download/libCube.so
#08 pc 00024491  /data/data/com.package.name/files/download/libCube.so
#09 pc 000244fb  /data/data/com.package.name/files/download/libCube.so
#10 pc 0003c159  /data/data/com.package.name/files/download/libCube.so
#11 pc 0003da23  /data/data/com.package.name/files/download/libCube.so
#12 pc 00016cd1  /data/data/com.package.name/files/download/libCube.so
#13 pc 00020b81  /data/data/com.package.name/files/download/libCube.so (InitHCDNDownloaderCreator+140)
#14 pc 0004671f  /data/data/com.package.name/files/download/libCube.so (Java_com_package_name_hcdndownloader_HCDNDownloaderCreator_InitCubeCreatorNative+322)
#15 pc 051d3453  /data/data/com.package.name/tinker/patch-2f63d418/odex/tinker_classN.dex
at com.package.name.HCDNDownloaderCreator.InitCubeCreatorNative(Native Method)
at com.package.name.HCDNDownloaderCreator.InitCubeCreator(Unknown Source)
at com.package.name.download.CubeLoadManager.b(Unknown Source)
............

#00  be837814  94ca06e8  /data/data/com.package.name/files/download/libHCDNClientNet.so
#01  be837818  0000002a
     be83781c  00792000
     be837820  be837874
     be837824  00000000
     be837828  00000000
     be83782c  b818ceec  [heap]
     be837830  00000000
     be837834  be837874
     be837838  be8378d0
     be83783c  b6f8a4dd  /system/bin/linker (__dl__ZN9ElfReader4LoadEPK17android_dlextinfo+44)
#02  be837840  00000000
     be837844  00000000
     be837848  00010300
     be83784c  b6f882f7  /system/bin/linker (__dl__ZL12find_libraryPKciPK17android_dlextinfo+582)
#03  be837850  00000000
     be837854  00000000
     be837858  00000000
     be83785c  00000000
............

............
943e8000-944e4000 rw-       0   fc000 [stack:8572]
944e4000-94c77000 r-x       0  793000 /data/data/com.package.name/files/download/libHCDNClientNet.so
94c77000-94ca1000 rw-  792000   2a000 /data/data/com.package.name/files/download/libHCDNClientNet.so
94ca1000-94cae000 ---       0    d000 
............
94eab000-94fa8000 rw-       0   fd000 [stack:8568]
94fa8000-9508f000 r-x       0   e7000 /data/data/com.package.name/files/download/libCube.so
9508f000-95094000 r--   e6000    5000 /data/data/com.package.name/files/download/libCube.so
95094000-95095000 rw-   eb000    1000 /data/data/com.package.name/files/download/libCube.so
95095000-95096000 rw-       0    1000 
............
b6f85000-b6f86000 r-x       0    1000 [sigpage]
b6f86000-b6f93000 r-x       0    d000 /system/bin/linker
b6f93000-b6f94000 r--    c000    1000 /system/bin/linker
b6f94000-b6f95000 rw-    d000    1000 /system/bin/linker
b6f95000-b6f96000 rw-       0    1000 
............
```

基本都出现在 Android 5.x 以及更低版本的设备上（这里的例子是发生在 5.x 上），某个动态库调用 linker 加载另一个动态库时发生了崩溃。

stack 和 maps 都太长了，只列了一部分。


# 分析

是 APP 自己的 bug ？还是特定 OS 版本或机型的兼容性问题？为什么 Android 6.x 及以上版本的系统中几乎没有这个崩溃? 有很多疑问。

经过分析，我们发现这三个现象的本质是一样的，这里仅针对现象 3 进行分析。

## ELF 信息和 maps 分析

通过机型匹配和 ELF build-id 匹配，拿到了和崩溃设备相同的 linker 二进制文件，另外两个动态库我们自己的服务器中有。所有 ELF 都是 armeabi 架构的。

linker:

```
$ arm-linux-androideabi-readelf -l ./linker

Elf file type is DYN (Shared object file)
Entry point 0xa18
There are 9 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00000034 0x00000034 0x00120 0x00120 R   0x4
  INTERP         0x000154 0x00000154 0x00000154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /system/bin/linker]
  LOAD           0x000000 0x00000000 0x00000000 0x0c444 0x0c444 R E 0x1000
  LOAD           0x00ca5c 0x0000da5c 0x0000da5c 0x00734 0x01c60 RW  0x1000
  DYNAMIC        0x00cef8 0x0000def8 0x0000def8 0x000c0 0x000c0 RW  0x4
  GNU_EH_FRAME   0x00c2e8 0x0000c2e8 0x0000c2e8 0x0015c 0x0015c R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0
  EXIDX          0x009060 0x00009060 0x00009060 0x00660 0x00660 R   0x4
  GNU_RELRO      0x00ca5c 0x0000da5c 0x0000da5c 0x005a4 0x005a4 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .dynsym .dynstr .hash .rel.dyn .rel.plt .plt .text .ARM.exidx .rodata .ARM.extab .eh_frame .eh_frame_hdr 
   03     .data.rel.ro.local .init_array .dynamic .got .data .bss 
   04     .dynamic 
   05     .eh_frame_hdr 
   06     
   07     .ARM.exidx 
   08     .data.rel.ro.local .init_array .dynamic .got
```

libCube.so:

```
$ arm-linux-androideabi-readelf -l ./libCube.so

Elf file type is DYN (Shared object file)
Entry point 0x0
There are 9 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00000034 0x00000034 0x00120 0x00120 R   0x4
  INTERP         0x000154 0x00000154 0x00000154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /system/bin/linker]
  LOAD           0x000000 0x00000000 0x00000000 0xe6af4 0xe6af4 R E 0x1000
  LOAD           0x0e6de8 0x000e7de8 0x000e7de8 0x04e44 0x0599c RW  0x1000
  DYNAMIC        0x0ea940 0x000eb940 0x000eb940 0x00108 0x00108 RW  0x4
  NOTE           0x000168 0x00000168 0x00000168 0x00024 0x00024 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0
  EXIDX          0x0b9e34 0x000b9e34 0x000b9e34 0x071a0 0x071a0 R   0x4
  GNU_RELRO      0x0e6de8 0x000e7de8 0x000e7de8 0x04218 0x04218 RW  0x8

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.gnu.build-id .dynsym .dynstr .hash .rel.dyn .rel.plt .plt .text .ARM.exidx .ARM.extab .rodata 
   03     .data.rel.ro.local .fini_array .init_array .data.rel.ro .dynamic .got .data .bss 
   04     .dynamic 
   05     .note.gnu.build-id 
   06     
   07     .ARM.exidx 
   08     .data.rel.ro.local .fini_array .init_array .data.rel.ro .dynamic .got
```

libHCDNClientNet.so:

```
$ arm-linux-androideabi-readelf -l ./libHCDNClientNet.so

Elf file type is DYN (Shared object file)
Entry point 0x0
There are 9 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00000034 0x00000034 0x00120 0x00120 R   0x4
  INTERP         0x000154 0x00000154 0x00000154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /system/bin/linker]
  LOAD           0x000000 0x00000000 0x00000000 0x792a40 0x792a40 R E 0x1000
  LOAD           0x792b10 0x00793b10 0x00793b10 0x28bd8 0x35ff0 RW  0x1000
  DYNAMIC        0x7af594 0x007b0594 0x007b0594 0x00100 0x00100 RW  0x4
  NOTE           0x000168 0x00000168 0x00000168 0x00024 0x00024 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0
  EXIDX          0x60fa90 0x0060fa90 0x0060fa90 0x26190 0x26190 R   0x4
  GNU_RELRO      0x792b10 0x00793b10 0x00793b10 0x1e4f0 0x1e4f0 RW  0x8

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.gnu.build-id .dynsym .dynstr .hash .rel.dyn .rel.plt .plt .text .ARM.extab .ARM.exidx .rodata 
   03     .data.rel.ro.local .fini_array .init_array .data.rel.ro .dynamic .got .data .bss 
   04     .dynamic 
   05     .note.gnu.build-id 
   06     
   07     .ARM.exidx 
   08     .data.rel.ro.local .fini_array .init_array .data.rel.ro .dynamic .got
```

对照崩溃时的 maps 信息，发现 libHCDNClientNet.so 的动态加载过程确实没有走完，第一个 LOAD Segment 被完全 mmap 到了内存，但是第二个 LOAD Segment 没有，只 mmap 了 `2a000` 长度，接着就崩溃了，从 maps 来看，并没有走到对 `.got` 等 section 做只读保护的阶段。

## linker 崩溃位置和内存数据分析

linker 调用 memset:

```
............
.text:00004050   LDR             R3, [R4,#0x18]
.text:00004052   LSLS            R3, R3, #0x1E
.text:00004054   BPL             loc_4068
.text:00004056   UBFX.W          R2, R10, #0, #0xC
.text:0000405A   CBZ             R2, loc_4068
.text:0000405C   MOV             R0, R10
.text:0000405E   MOVS            R1, #0
.text:00004060   RSB.W           R2, R2, #0x1000
.text:00004064   BLX             __dl_memset
.text:00004068   ADDW            R1, R10, #0xFFF
.text:0000406C   BIC.W           R10, R1, #0xFF0
.text:00004070   BIC.W           R0, R10, #0xF
............
```

memset 中发生了 SIGBUS 崩溃：

```
............
.text:000047B0 __dl_memset
.text:000047B0 ; __unwind {
.text:000047B0   STMFD           SP!, {R0}
.text:000047B4   CMP             R2, #0x10
.text:000047B8   BCC             loc_4890
.text:000047BC   MOV             R3, R0
.text:000047C0   MOV             R1, R1,LSL#24
.text:000047C4   ORR             R1, R1, R1,LSR#8
.text:000047C8   ORR             R1, R1, R1,LSR#16
.text:000047CC   ANDS            R12, R3, #7
.text:000047D0   BNE             loc_4868
.text:000047D4   MOV             R0, R1
.text:000047D8   SUBS            R2, R2, #0x40
.text:000047DC   BCC             loc_480C
.text:000047E0   STRD            R0, [R3]  ;在这个位置发生了 SIGBUS
.text:000047E4   STRD            R0, [R3,#8]
.text:000047E8   STRD            R0, [R3,#0x10]
.text:000047EC   STRD            R0, [R3,#0x18]
.text:000047F0   STRD            R0, [R3,#0x20]
.text:000047F4   STRD            R0, [R3,#0x28]
.text:000047F8   STRD            R0, [R3,#0x30]
.text:000047FC   STRD            R0, [R3,#0x38]
.text:00004800   ADD             R3, R3, #0x40
.text:00004804   SUBS            R2, R2, #0x40
.text:00004808   BGE             loc_47E0
............
```

`R0` 的值是 `0`，`R3` 的值是 `94ca06e8`，这里试图将 `0` 写入虚拟内存地址为 `94ca06e8` 的内存中。`94ca06e8` 位于之前提到的 libHCDNClientNet.so 中 “不完整的第二个 LOAD Segment 的 mmap 内存映射区域” 中：

```
94c77000-94ca1000 rw-  792000   2a000 /data/data/com.package.name/files/download/libHCDNClientNet.so
```

这个 Segment 的 ELF 信息：

```
Type  Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
LOAD  0x792b10 0x00793b10 0x00793b10 0x28bd8 0x35ff0 RW  0x1000
```

由于需要 4K 对其，我们看到这个 Segment 是从文件 Offset `792000` 开始映射的，映射到虚拟内存地址 `94c77000`，所以实际映射到内存的文件长度是：`792b10 - 792000 + 28bd8 = 296e8`，对应的虚拟内存区间是：`[94c77000, 94ca06e8)`。而 memset 发生崩溃时正试图向 `94ca06e8` 指向的内存中写入数据 `0`。

memset 之类的 libc/bionic 函数由于功能比较明确单一，一般都经过了很严格的测试，发生问题的概率很小。这里也比较明显是调用 memset 时指定的内存写入地址有问题。

## AOSP 5.x 源码分析

崩溃位置的 AOSP 5.x 源码：

```cpp
bool ElfReader::LoadSegments() {
  for (size_t i = 0; i < phdr_num_; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table_[i];
    
    if (phdr->p_type != PT_LOAD) {
      continue;
    }
    
    // Segment addresses in memory.
    ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
    ElfW(Addr) seg_end   = seg_start + phdr->p_memsz;
    
    ElfW(Addr) seg_page_start = PAGE_START(seg_start);
    ElfW(Addr) seg_page_end   = PAGE_END(seg_end);
    
    ElfW(Addr) seg_file_end   = seg_start + phdr->p_filesz;
    
    // File offsets.
    ElfW(Addr) file_start = phdr->p_offset;
    ElfW(Addr) file_end   = file_start + phdr->p_filesz;
    
    ElfW(Addr) file_page_start = PAGE_START(file_start);
    ElfW(Addr) file_length = file_end - file_page_start;
    
    if (file_length != 0) {
      void* seg_addr = mmap(reinterpret_cast<void*>(seg_page_start),
                            file_length,
                            PFLAGS_TO_PROT(phdr->p_flags),
                            MAP_FIXED|MAP_PRIVATE,
                            fd_,
                            file_page_start);
      if (seg_addr == MAP_FAILED) {
        DL_ERR("couldn't map \"%s\" segment %zd: %s", name_, i, strerror(errno));
        return false;
      }
    }
    
    // if the segment is writable, and does not end on a page boundary,
    // zero-fill it until the page limit.
    if ((phdr->p_flags & PF_W) != 0 && PAGE_OFFSET(seg_file_end) > 0) {
      //这里调用 memset，之后发生了崩溃。
      memset(reinterpret_cast<void*>(seg_file_end), 0, PAGE_SIZE - PAGE_OFFSET(seg_file_end));
    }    
............
```

Google Source 上的对应源码在 [这里](https://android.googlesource.com/platform/bionic/+/refs/tags/android-5.0.2_r3/linker/linker_phdr.cpp#374)。

简单分析后发现并没有问题。这里将 LOAD Segment 用 mmap 映射到内存后，如果发现该 Segment 对应的映射内存是可写的，且结尾处不是刚好 4K 对齐的，就将最后 4K 内存中剩余的未映射部分用 memset 填 `0`，虽然末尾的这个区域可能没有物理文件与之对应，但是在这个区域执行写入并不会触发 SIGBUS，见 [mmap manpage](http://man7.org/linux/man-pages/man2/mmap.2.html)：

> A file is mapped in multiples of the page size. For a file that is not a multiple of the page size, the remaining memory is zeroed when mapped, and writes to that region are not written out to the file.

反之，如果是由于这个原因导致 SIGBUS，那就不是千分之一的崩溃概率了，只要可写 Segment 的结尾处不是 4K 对齐的，linker 的代码就会走到这里，就必定会崩溃。

## AOSP 6.x 源码分析

从线上的数据统计来看，这个崩溃 99% 以上发生在 Android 5.x 及以下系统中。那么 Android 6.x linker 做了什么呢？

```cpp
bool ElfReader::LoadSegments() {
  for (size_t i = 0; i < phdr_num_; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table_[i];
    
    if (phdr->p_type != PT_LOAD) {
      continue;
    }
    
    // Segment addresses in memory.
    ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
    ElfW(Addr) seg_end   = seg_start + phdr->p_memsz;
    
    ElfW(Addr) seg_page_start = PAGE_START(seg_start);
    ElfW(Addr) seg_page_end   = PAGE_END(seg_end);
    
    ElfW(Addr) seg_file_end   = seg_start + phdr->p_filesz;
    
    // File offsets.
    ElfW(Addr) file_start = phdr->p_offset;
    ElfW(Addr) file_end   = file_start + phdr->p_filesz;
    
    ElfW(Addr) file_page_start = PAGE_START(file_start);
    ElfW(Addr) file_length = file_end - file_page_start;
    
    if (file_size_ <= 0) {
      DL_ERR("\"%s\" invalid file size: %" PRId64, name_, file_size_);
      return false;
    }
    
    if (file_end >= static_cast<size_t>(file_size_)) {
      DL_ERR("invalid ELF file \"%s\" load segment[%zd]:"
          " p_offset (%p) + p_filesz (%p) ( = %p) past end of file (0x%" PRIx64 ")",
          name_, i, reinterpret_cast<void*>(phdr->p_offset),
          reinterpret_cast<void*>(phdr->p_filesz),
          reinterpret_cast<void*>(file_end), file_size_);
      return false;
    }
    
    if (file_length != 0) {
      void* seg_addr = mmap64(reinterpret_cast<void*>(seg_page_start),
                            file_length,
                            PFLAGS_TO_PROT(phdr->p_flags),
                            MAP_FIXED|MAP_PRIVATE,
                            fd_,
                            file_offset_ + file_page_start);
      if (seg_addr == MAP_FAILED) {
        DL_ERR("couldn't map \"%s\" segment %zd: %s", name_, i, strerror(errno));
        return false;
      }
    }
    
    // if the segment is writable, and does not end on a page boundary,
    // zero-fill it until the page limit.
    if ((phdr->p_flags & PF_W) != 0 && PAGE_OFFSET(seg_file_end) > 0) {
      memset(reinterpret_cast<void*>(seg_file_end), 0, PAGE_SIZE - PAGE_OFFSET(seg_file_end));
    }
............
```

Google Source 上的对应源码在 [这里](https://android.googlesource.com/platform/bionic/+/refs/tags/android-6.0.0_r11/linker/linker_phdr.cpp#385)。

我们确实看到了显著的区别，从 Android 6.0 开始，执行 mmap 之前增加了额外的检查，如果映射的文件结尾 offset 大于等于了文件的长度，就直接 `return false`。这会在 native 层表现为 `dlopen()` 返回失败，在 java 层表现为 `System.loadLibrary()` 抛出异常。

查看 Android 源码的 git log 我们发现：2015 年 6 月 26 日，Dmitriy Ivanov 将这个问题的修复 [patch](https://android-review.googlesource.com/c/platform/bionic/+/156745) merge 到了 bionic 的 master 分支。commit message 为："Fix crash when trying to load invalid ELF file.”。至此，这个问题得到了修复。


# 结论

## 崩溃原因

Android 6.0 之前版本的 linker 完全信任了目标 ELF 中的各种基本信息。当 ELF 文件的头部完整（能解析出 Program Headers 等信息）但文件本身有缺失时，可能会导致某个可写 LOAD segment 缺失了一部分或全部，只要缺失长度超过 4K，就会导致 [mmap manpage](http://man7.org/linux/man-pages/man2/mmap.2.html) 中提到的条件不再满足：

> A file is mapped in multiples of the page size. For a file that is not a multiple of the page size, the remaining memory is zeroed when mapped, and writes to that region are not written out to the file.

当缺失长度超过 4K，后续 memset 的写入尝试就会导致 SIGBUS。如 [mmap manpage](http://man7.org/linux/man-pages/man2/mmap.2.html) 中提到的那样：

> SIGBUS: Attempted access to a portion of the buffer that does not correspond to the file (for example, beyond the end of the file, including the case where another process has truncated the file).

引起崩溃的原因已经很清楚了：**linker 加载的 ELF 文件不完整，缺失了后面的一部分**。

进一步分析线上的数据，发现 linker 只在加载动态下发的动态库时发生这个崩溃，而加载安装包内的动态库时几乎从来没有遇到过这个问题，所以基本可以断定是在动态库的下载/解压/校验的某个环节出了问题。

## 验证

我们可以很方便的在 Linux 上写一小段 C 代码进行验证：

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

#define MAP_SZ       8192
#define MAP_FILE_SZ  (4096 + 10)
#define REAL_FILE_SZ (4096 + 10)

int main(int argc, char *argv[])
{
    int fd = open("./data", O_RDWR | O_CREAT | O_TRUNC, 0644);
    ftruncate(fd, REAL_FILE_SZ);

    void *addr = mmap(NULL, MAP_SZ, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);
    memset(addr + MAP_FILE_SZ, 0, MAP_SZ - MAP_FILE_SZ);
    printf("memset OK!\n");

    munmap(addr, MAP_SZ);
    close(fd);
    
    printf("everything OK!\n");
    return 0;
}
```

这里创建了一个 4K + 10 字节长度的文件，按照 mmap 的页对齐要求，指定分配了 8K 长度的内存空间，用 memset 向 `addr[4K + 10, 8K)` 内存空间写入数据。最后生成了一个 4K + 10 字节长度的文件。向 `addr[4K + 10, 8K)` 内存空间的写入操作没有产生任何副作用，没有引起崩溃：

```
caikelun@debian:~/test$ gcc ./test.c
caikelun@debian:~/test$ ./a.out
memset OK!
everything OK!
caikelun@debian:~/test$ ls -al
-rw-r--r--  1 caikelun caikelun  4106 May 31 18:06 data
```

把代码稍作修改，将实际文件长度从 4K + 10 字节改为 4K - 10 字节，其他不变：

```c
#define REAL_FILE_SZ (4096 + 10)
```

改为：

```c
#define REAL_FILE_SZ (4096 - 10)
```

再次编译和执行程序。运行到 memset 时发生了 SIGBUS：

```
caikelun@debian:~/devel/test$ gcc ./test.c
caikelun@debian:~/devel/test$ ./a.out
Bus error
caikelun@debian:~/test$ ls -al
-rw-r--r--  1 caikelun caikelun  4086 May 31 18:10 data
```

# 关于崩溃捕获工具

因为这是本系列的第一篇，在这里介绍一下我们一直在使用的自己开发的 Android APP 崩溃捕获工具: [xCrash](https://github.com/iqiyi/xCrash)。

完整准确的 backtrace 对于定位线上崩溃问题很重要。寄存器、stack、maps、FD list、内存统计、各种 log、甚至 ELF build-id 信息也同样重要。

有些线上的系统相关 native 崩溃，我们竭尽全力的去获取崩溃现场的各种信息，却依然会感到信息不足。为了获取更多需要的信息，我们只能冒着 “xCrash 本身发生崩溃” 的风险，小心翼翼的去改进。当 xCrash 被唤起运行时，经常要面对这样的情况：栈已经溢出、堆内存不可用、FD完全耗尽、flash空间完全耗尽、正在崩溃中的进程随时会被系统强杀。这些极端的情况并非是我们自虐式的臆想，而是线上崩溃都遇到过的真实情况。有时，这些极端情况的出现，本身就是导致进程崩溃的间接原因。

如果线上 xCrash 本身发生了崩溃，没有人能告诉我们崩溃在哪里？为什么崩溃？如果 xCrash 无法如预期的运行拿到所有需要的信息，也没有人能告诉我们为什么？因为这是最后一道防线，下一刻，等待 APP 进程的就是所有资源的回收，一切归零。
