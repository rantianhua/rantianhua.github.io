---
layout: post
title:  "Dalvik 虚拟机和 ART 虚拟机的区别"
date:   2019-04-01 19:39:00
categories: Android
---

# Dalvik 虚拟机

## 简介

是由 Google 等厂商合作开发的 Android 移动设备平台的核心组成部分之一，可以支持转换为 DEX 格式的 Java 应用程序的运行。其中 DEX 格式就是专门为 Dalvik 虚拟机设计的一种压缩格式，适合内存和处理器速度有限的系统。

## 特性

- Dalvik 经过优化后，允许在有限的内存中运行多个虚拟机实例。在 Android 中每个应用程序都是一个独立的虚拟机进程。
- Dalvik 虚拟机中寄存器都是 32 位的。
- Dalvik 最多支持 65536 个寄存器。
- 使用 JIT 编译技术（具体后面讲解）。

# ART 虚拟机

## 简介

ART 是 Android Runtime 的缩写。是由 Google 公司研发的新一代虚拟机，在 2013 年 Android 4.4 系统发布的同时发布了 ART 的预览版，最终在 Android 5.0 上正式发布，取代了以往的 Dalvik 虚拟机。

## 特性

- 使用 AOT 编译技术（具体后面讲解）

# 区别

## 编译技术角度

### JIT 介绍

Android 系统发布之初，Dalvik 通过处理 dex/odex 文件来运行 Java 应用程序，从 Android 2.2 Froyo 开始，Google 在 Dalvik 虚拟机中加入了 Just-In-Time Compiler ，即 JIT 编译器。其主要工作是对执行次数频繁的 dex/odex 代码进行编译与优化，将 dex 格式的字节码翻译成精简的 Native Code 去执行，将 Dalvik 的性能提升了 3~6 倍。

### AOT 介绍

AOT (Ahead-of-time) 是随着 ART 虚拟机开始发布于 Android 4.4 KitKat 。其主要工作是在应用安装的时候启动 dex2oat 过程把 dex 预编译成 ELF 可执行文件，这样就将应用程序变成了本地应用，不需要每次运行的时候都进行重新编译。其整个过程与 Dalvik 的 JIT 对比如下：

![JIT 和 AOT 工作流程对比图]({{site.baseurl}}/images/dalvik-art-process.png) 

### 总结

先回顾 JIT 编译器，每次程序启动的时候都会进行 JIT 过程，这也从它的名字体现出来了，它实际是一个动态的重复的过程。这页导致运行时比较耗电，整个过程如下所示：

![JIT 编译过程]({{site.baseurl}}/images/jit-process.png)

与 JIT 不同的是，AOT 在应用安装期间执行，将 Java 应用变成了本地应用，运行时相比之前没有多余的解释过程，所以运行地更流畅了，同时解决了 JIT 带来的耗电问题。但同时，其静态执行的特性导致安装过程变慢，还需要更多存储空间存储 ELF 文件。其整个过程如下：

![AOT 编译过程]({{site.baseurl}}/images/aot-process.png)

事实上，从 Android 7.0 Nougat 开始，为了进一步解决 AOT 引来的新问题，JIT 重新回归，和 AOT 一起进一步改善 Android 应用程序安装以及运行的性能：

![AOT/JIT 混合模式]({{site.baseurl}}/images/aot-and-art.png)

在这种模式下，应用在安装时不进行 dex2oat 过程，所以安装依然变快了。运行时 dex/odex 文件解释执行，在这个过程中，虚拟机会识别出热点函数，将 JIT 运用在这些热点函数上，编译后的产物存储在 JIT Code Cache 并生成 profile 文件记录热点函数信息。当手机在空闲的时候，系统扫描 App 目录下的 profile 文件执行 AOT 过程。如此一来，运行时的性能以及耗电问题也得到了解决。

## 内存角度

### Dalvik 堆内存结构

![Dalvik 内存结构]({{site.baseurl}}/images/dalvik_memory_struct.png)

Dalvik 虚拟机将运行时堆划分为三个区块：

- LinearAlloc : 一个只读的线性内存空间，主要用来存储虚拟机中的类等一些在整个进程的生命周期中都不能结束的永久数据。
- Zygote Space : 用来管理 Zygote 进程启动过程中预加载和创建的各种对象。
- Allo Space : Zygote 进程在 fork 第一个子进程之前，把还没有使用的那部分划分为 Allocation Space ，无论 Zygote 进程还是应用进程，都是在这个区域上分配内存。

### ART 堆内存结构

![ART 内存结构]({{site.baseurl}}/images/art_memory_struct.png)

相比 Dalvik 多了两个区块：

- Image Space : 与 LinearAlloc Space 类似，用来存放应用程序的一些预加载资源。
- Large Object Space : 一些离散地址的集合，用来分配一些大对象。

### 总结

相比 Dalvik ，ART 新增两个内存块，提升了内存分配的效率，并有效减少了碎片的产生，同时对 GC 也产生了正面影响。

## GC 角度

### GC 的类型

- GC_CONCURRENT : 当应用进程中的 Heap 内存占用上涨时，避免因为 Heap 内存满了而触发 GC 。
- GC_FOR_MALLOC : 由于 Concurrent GC 没有及时执行完，应用又需要分配更多内存，此时需要停下来进行 Malloc GC 。
- GC_EXTERNAL_ALLOC : 分配的 Native 内存执行的 GC 。
- GC_HPROF_DUMP_HEAP : 创建一个 HPROF profile 的时候执行。
- GC_EXPLICIT : 显示调用了 `System.gc()` 的时候执行。

### ART 所做的优化

* 只有一次（而非两次）GC 暂停，Dalvik 中第一次暂停是标记当前线程的根对象（分配在栈中的对象和全局对象）。
* 在 GC 保持暂停状态期间并行处理，即在 Dalvik 中进行的弱引用清楚、重新标记非线程根和卡片预清理。
* 在清理最近分配的短时对象这种特殊情况中，回收器的总 GC 时间更短（因为粘性 CMS 是从上次 GC 后新分配的对象里进行 GC 操作）。
* 优化了垃圾回收的工效，能够更加及时地进行并行垃圾回收，这使得 GC_FOR_ALLOC 事件在典型用例中极为罕见。
* 压缩 GC 以减少后台内存使用和碎片。

# 参考

> https://juejin.im/post/5c232907f265da61662482b4

> https://source.android.com/devices/tech/dalvik
