---
layout: post
title:  "FileDownloader VS OkDownload - 消息流系统篇"
date:   2019-03-04 12:13:00
categories: Android
---

# 前言
[FileDownloader](https://github.com/lingochamp/FileDownloader) 和 [OkDownload](https://github.com/lingochamp/okdownload) 都是十分优秀的且应用广泛的下载引擎。其中后者可以看做是前者的 2.0 版，相较于前者，后者在设计上更加轻量，更加注重单元测试，更加注重性能优化。这篇文章将围绕消息流系统对两个库分别阐述。让大家了解到前后设计上的优化过程。

# FileDownloader 中的消息流系统

`FileDownloader` 中的状态由 `FileDownloadStatus` 描述，这些状态包括：

* __toLaunchPool__ 表示 Task 即将启动
* __toFileDownloadService__ 表示 Task 已经发送到 FileDownloadService 准备开始下载
* __pending__ 表示 Task 在 FileDownloadService 的下载列表里
* __started__ 表示当前正在处理该 Task
* __connected__ 表示 Task 的文件大小已经通过网络确认了
* __progress__ 表示正在通过 Socket 流下载文件
* __blockComplete__ 表示下载已经完成，不过在回调 completed 之前，可以在该回调里处理类似于解压的操作，只有该回调运行在异步线程，其他都是运行在主线程
* __retry__ 表示 Task 由于出错正准备重试
* __error__ 表示下载失败
* __paused__ 表示任务暂停了
* __completed__ 表示任务完成了
* __warn__ 表示有同样的 Task 正在下载

上面这些状态通过 `FileDownloadListener` 进行回调，整个回调流程如下图所示：

![FileDownloadListener 的回调流程](http://upload-images.jianshu.io/upload_images/219854-f79419c81782f51a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时展示一下整个的的消息流向：

![消息流向图](http://upload-images.jianshu.io/upload_images/219854-86b444ac4166491c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

`MessageSnapshotTaker` 先生成一个 message 的快照信息，通常里面会包含文件当前已经下载的进度、文件的总进度、异常信息（如果是下载过程中出现异常的话）、Task ID，而 message 本身就代表了各个状态。

生成的消息传递给 `MessageSnapshotFlow` ，它有两种方式把消息发送给 `MessageReceiver` ：同步和异步。在以下两种情况下：
1. 在开始一个任务前检查到该任务已经下载完成了，那么直接回调 completed。
2. 在开始一个任务前检查到该任务是一个重复的任务，那么直接回调 warn 。
会用同步的方式，其他均按照异步的方式。

`MessageSnapshotFlow` 会创建一个数量为 5 的线程池列表， 其中每个线程池的线程数为 1 。该线程池列表的作用是保证每个 task 都对应一个线程池去发送它的消息，这样消息能按照 FIFO 的顺序发出。

`MessageSnapshotFlow` 发出的消息由 `MessageReceiver` 接收，如图所示，它有两个对应的实现：
1. `MessageSnapshotGate`：运行在主进程，它会先根据 message 里面的 Task ID 找出对应的 Task ，然后根据当前接收的 message 的状态和此刻 Task 的状态对 message 进行过滤 ，过滤后的 message 将被发送到 `DownloadTaskHunter` 进行处理。
2. `FDServiceSeparateHandler` ：运行在 `filedownloader` 进程，在它收到 message 后，会通过 IPC 调用最终把 message 发送给 `MessageSnapshotGate` 。

关于消息的过虑，可以用一张图来描述一下：

![消息过滤图](http://upload-images.jianshu.io/upload_images/219854-93c9f8e6642ae391?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

箭头的指向表示各个 status 可以发生转变的路径，图中第一个箭头右边为×表示此类 status 转变是无效的，需要过滤掉，其他不符合箭头中指定的规则的转变也会被过滤掉。

`DownloadTaskHunter` 收到消息后，会统计、收集 message 里面携带的信息，比如记录文件总大小，更新当前下载进度，计算下载速度等等。然后消息会被存储在 `FileDownloadMessenger`  的一个  FIFO 的队列中，接着通过 `FileDownloadMessageStation` 进行回调速率的控制，如下图所示：

![速率控制图](http://upload-images.jianshu.io/upload_images/219854-a7f8ae80e240ab9d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`FileDownloadMessageStation` 要控制的对象是 `IFileDownloadMessenger` ，每个 Task 都会有这样一个对象来处理消息回调。它的内部维护着两个列表，其中一个是 waitingQueue ，它用来存储接收的 `IFileDownloadMessenger` ，
存储完成后会进行检查，看当前是否有 `IFileDownloadMessenger` 正在处理它的消息，如果没有，那么把一定数量的 `IFileDownloadMessenger` 转移到  另一个列表 `disposingList` （它的大小默认是 5 ，可以通过 `FileDownloader#setGlobalHandleSubPackageSize(int)` 进行设置），然后等待一定得时间间隔（默认是 10 毫秒，可以通过 `FileDownloader#setGlobalPost2UIInterval(int)` 方法进行设置）后通知 `diposingList` 里面的每个 `IFileDownloadMessenger ` 进行消息回调。其中等待时间间隔的操作是通过一个 UI Handler 的 postDelay 的方式实现。

消息回调的过程就比较简单了，从 `FileDownloadMessenger` 内部的队列头部拿出消息，然后按照消息类型回调 `FileDownloadListener` 对应的接口。

---

# OkDownload 的消息流系统

OkDownload 的状态包括：
* PENDING
* RUNNING
* COMPLETED
* IDLE
* UNKNOWN

可以很明显地感觉到，相比于 `FileDownloader` ，这里的状态精简了很多。其实在 `FileDownloader` 中，状态确实比较冗余。从根本上来讲，这跟整个下载的架构设计相关，架构设计得越复杂，状态切换就会越多，反之状态就能得到精简。

`OkDownload` 的各个状态通过 [DownloadListener](https://github.com/lingochamp/okdownload/blob/master/okdownload/src/main/java/com/liulishuo/okdownload/DownloadListener.java) 进行回调。在整个的下载过程中，`DownloadListener` 的回调过程可以用下图来描述：

![DownloadListener 回调流程](http://upload-images.jianshu.io/upload_images/219854-9b4f4ac84f20936d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先从接口的名称上看，和 `FileDownloadListener` 有了很大的差别。 `DownloadListener` 将试探连接（用来获取文件大小的连接）的开始和结束、下载的起点（从断点出下载还是从开头下载）都通过接口暴露出去，这在 `FileDownloader` 中没有的。另外一个很大的变化在于整个流程的尾部。在 `FileDownloader` 的流程图里我们可以看到有四个回调，而在 `OkDownloadListener` 里面只有一个回调。这在使用上能带来很大方便，很多时候我们只关心结束点，但在 `FileDownloader` 里面，我们需要处理四个地方，而在 `OkDownload` 里面处理一个地方就好了。

还有很重要的一个改变是，`OkDownload` 将下载块儿做得更加细粒度化，每个下载块儿的 connect / fetch / end 都可以回调出去，对用户来说，他们能更加清晰当前的下载状况。而在 `FileDownloader` 里，只有一个 retry 和总的 progress 回调，分片过程对用户来说是不可感知的。

综合来看，`DownloadListener` 的接口和整个下载过程贴合地十分紧密，基本上通过这些回调就能准确知道下载引擎内部在干什么，进行到哪一步了。

现在回到消息流系统，在分析 FileDownloader 的时候，这是最主要的最复杂的部分，而其实到了 OkDownload 这里，一切都得到了简化。我们先来看看 FileDownloader 的消息流系统存在哪些问题：
1. 消息的路径太长了，从消息产生，差不多要经历 7 次传递，才能回调给用户。
2. `MessageSnapShotFlow` 在把消息传出去的时候，使用了一个线程池列表，复杂度肯定会提升，其实没有什么特别的必要去使用这样一个线程池列表。
3. 复杂的消息过滤流程。
4. 速率控制其实只要针对 progress 回调即可，因为也只有该回调才会出现十分密集的情况。

 `OkDownload` 的消息流处理其实十分简单，全部交由 `CallbackDispatcher` 。它以单例的形式存在，消息源就来自下载过程中的各个组件，内部有两个工具辅助，一个是 UI Handler ，如果该 Task 要求所有的回调都在主进程，那么 `CallbackDispatcher` 收到的所有消息都通过 Handler 发送到主线程，然后直接通过 Task 获取到它的 `DownloadListener` ，紧接着进行对应回调即可；另一个是内部封装的 `DownloadListener` ，由它直接代理给真正的 Listener 进行回调，中间没有任何的线程切换。之所以有个代理是因为 `DownloadMonitor` 这个全局监听器需要进行监听工作。

而速率控制只在回调 progress 的时候会有，控制的逻辑也很简单，用户在配置 Task 的时候，可以通过 `setMinIntervalMillisCallbackProcess(int)` 方法设置 progress 回调的最小时间间隔，该间隔就是回调的速率。

可以看到，OkDownload 的整个消息流的处理相比 FileDownloader 简化了很多：回调流程中至多有一个线程切换，省掉了线程池；本身的状态切换很少，且任务出口只有一个，所以消息过滤其实被去掉；整个消息流的路径也是简单直接；速率控制也得到很大简化。

不过，OkDownload 所做的简化不仅仅如此。事实上， `DownloadListener` 本身的接口还是很多的，而在实际使用中，很多时候只关注那么几个接口，有些时候又只关注另外几个接口。考虑到这些 case ，OkDownload  对 Listener 做了很多封装，具体可以看到下图：

![DownloadListener1 and DownloadListener2](http://upload-images.jianshu.io/upload_images/219854-9bec5f2809578f06?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![DownloadListener3 and DownloadListener4](http://upload-images.jianshu.io/upload_images/219854-b951a1617244c96b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过这些封装的 Listener ，你可以更方便简洁地使用回调。因为在 OkDownload 中所有的结束状态，无论是正常下载完成还是因为出错了，都是统一一个回调且在内部保证了是最后一个回调，所以没有了多余的过滤过程。