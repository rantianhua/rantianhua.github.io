---
layout: post
title:  "从 FileDownloader 到 OkDownload 的无缝切换"
date:   2019-11-08 11:54:00
categories: Android
---

[FileDownloader](https://github.com/lingochamp/FileDownloader) 是 Android 平台下一个优秀的下载引擎，它主要有以下优点：
- 简单易用
- 支持独立进程
- 单任务分块下载
- 高并发支持
- 断点续传支持

这些优点使得 FileDownloader 非常受欢迎。然而，FileDownloader 本身不够轻量，且单测覆盖率十分之低。并且，前面说的独立进程的支持，随着 Android 8 系统开始对后台服务的限制越来越严格，从一个优点慢慢变成了缺点。为了兼容 Android 8 及之后的系统，在后台开启下载任务时，`FileDownloadService` 必须成为一个前台 Service ，用户会看到一个表示下载正在进行的通知。FileDownload 1.7.6 版本开始，我也提供了通知的定制化组件，具体可以参考[这里](https://github.com/lingochamp/FileDownloader/wiki/Compatibility-of-Android-O-Service)。可是，在一个公共 library 去做系统兼容方便的事情，会显得十分吃力，往往会做很多很糟糕的妥协。最可怕的事，往往有些问题在 library 的层面是无法解决的，比如 [Issue 1209](https://github.com/lingochamp/FileDownloader/issues/1209) 。

为了之后开发新的功能以及维护工作，我们开发了新的下载引擎 [OkDownload](https://github.com/lingochamp/okdownload) 。它拥有 FileDownloader 的所有优点同时还具有很多 FileDownloader 不具备的优点。具体不在这里赘述，大家可以参考 [Wiki](https://github.com/lingochamp/okdownload/wiki/Why-choose-OkDownload) 。

接下来着重说明怎么进行无缝切换。OkDownload 是十分轻量的，为了大家已经写好的 FileDownloader 的代码依然可用，为 OkDownload 提供了一个新的子 library 叫 filedownloader 。引入该库，之前的代码不用做任何改动依然生效，新的下载使用 OkDownload 的 API 即可。

```gradle
dependencies {
    // implementation "com.liulishuo.filedownloader:library:1.7.7"
    implementation "com.liulishuo.okdownload:okdownload:1.0.7-SNAPSHOT"
    implementation "com.liulishuo.okdownload:filedownloader:1.0.7-SNAPSHOT"
    implementation "com.liulishuo.okdownload:sqlite:1.0.7-SNAPSHOT"
}
```
所谓无缝切换的目的，上面的例子其实已经做到了，只是换了一个 library 的依赖即可。大部分情况下，应该就没有问题了。但是有些情况还是需要向大家说明。FileDownloader 和 OkDownload 都采用数据库的方式记录任务的断点信息。但是，FileDownloader 和 OkDownload 的数据表以及对应的数据结构区别已经非常大了，在 OkDownload 里面去适配 FileDownloader 的数据库是非常繁重且收效甚微的事情。而且，在 OkDownload 里面已经不使用临时文件的方式了，所以就算费很大力把数据迁移到 OkDownload ，文件层面也没法复用之前的数据。换言之，迁移到 OkDownload 后，之前 FileDownlaoder 还未下载完成的那些任务的断点信息就没有了。事实上这也不是什么大问题，OkDownload 会重新下载这些任务，且之后的断点功能不会有任何影响。在 OkDownload 里，为 `FileDownloader` 提供了一个方法 `discardFileDownloadDatabase` 可以清除旧的 FileDownloader 的数据库。如果你在使用 FileDownloader 的时候自定义了 `FileDownloadDatabase`，那么这个自定义数据库也不会再生效。

FileDownloader 还提供了 `IdGenerator` 的自定义，但在 OkDownload 里面已没有该组件的支持。OkDownload 中同一任务的 id 在不同时间启动是会变化的。所以，如果你的一些逻辑依赖于对任务 id 的持久化（比如存到数据库），那么这个切换到 OkDownload 之后，该部分逻辑会有问题。在 FileDownloader 的 Demo 中，提供了一个 `TasksManagerDemoActivity` 样例展示如何管理多个任务。这个例子里面就有我前面说的逻辑。需要做的调整也是比较简单的，我单独提了个 [Pull Request](https://github.com/lingochamp/FileDownloader/pull/1297) 说明在纠正对应逻辑时应该怎么做。

FileDownloader 中的一些方法在 OkDownload 中被 Deprecated 或者删除了，不过绝大部分应该不影响大家的实际使用。希望大家使用 OkDownload 1.0.7-SNAPSHOT 去尝试从 FileDownloader 切换到 OkDownload ，过程中有任何问题欢迎在 OkDownload 里面提 Issue 。