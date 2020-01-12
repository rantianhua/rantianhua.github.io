---
layout: post
title:  "Android AIDL 的使用及原理"
date:   2018-01-01 20:04:00
categories: Android
---

# 概述

Android 系统在 Linux 基础上增加了一种新的进程间通信的方式：Binder 机制。这是 Android 系统及其重要的组成部分。Binder 机制本身是一个跨越 native 层和 framework 层的通信机制，日常开发中是通过 AIDL 来使用底层的 Binder 机制。

利用 AIDL 可以定义客户端与服务端均认可的编程接口，接口描述的一系列操作无法直接在两个进程之间进行交互，需要将这些操作分解成可供操作系统理解的原语。这个分解以及组合的过程非常繁琐，AIDL 将其进行了简化。

本片文章将以 [FileDownloader](https://github.com/lingochamp/FileDownloader) 为例说明如何使用 AIDL 实现 Binder 通信以及 Java 层的实现原理。

## 场景描述

FileDownloader 的核心逻辑默认运行在独立进程，但是启动下载、暂停下载这些操作是在主进程调用的，实际这就是一个跨进程通信的过程，需要用到 Binder 机制。

## 创建 AIDL 文件

### IFileDownloadIPCService.aidl

```java
package com.liulishuo.filedownloader.i;

import com.liulishuo.filedownloader.model.FileDownloadHeader;
import android.app.Notification;

interface IFileDownloadIPCService {
    void start(
        String url, 
        String path, 
        boolean pathAsDirectory, 
        int callbackProgressTimes,
        int callbackProgressMinIntervalMillis, 
        int autoRetryTimes, 
        boolean forceReDownload,
        in FileDownloadHeader header, 
        boolean isWifiRequired);
    boolean pause(int downloadId);
    ...
}
```

### FileDownloadHeader.aidl

```java
package com.liulishuo.filedownloader.model;

parcelable FileDownloadHeader;
```

### 生成 java 文件

通过命令 `./gradlew clean build` 或者 `aidl` 命令可以生成对应的 .java 文件。上述       `IFileDownloadIPCService.aidl` 将会生成 `IFileDownloadIPCService.java` 文件：

```java
package com.liulishuo.filedownloader.i;

public interface IFileDownloadIPCService extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.liulishuo.filedownloader.i.IFileDownloadIPCService {
        private static final java.lang.String DESCRIPTOR = "com.liulishuo.filedownloader.i.IFileDownloadIPCService";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.liulishuo.filedownloader.i.IFileDownloadIPCService interface,
         * generating a proxy if needed.
         */
        public static com.liulishuo.filedownloader.i.IFileDownloadIPCService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.liulishuo.filedownloader.i.IFileDownloadIPCService))) {
                return ((com.liulishuo.filedownloader.i.IFileDownloadIPCService) iin);
            }
            return new com.liulishuo.filedownloader.i.IFileDownloadIPCService.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_start: {
                    data.enforceInterface(descriptor);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    java.lang.String _arg1;
                    _arg1 = data.readString();
                    boolean _arg2;
                    _arg2 = (0 != data.readInt());
                    int _arg3;
                    _arg3 = data.readInt();
                    int _arg4;
                    _arg4 = data.readInt();
                    int _arg5;
                    _arg5 = data.readInt();
                    boolean _arg6;
                    _arg6 = (0 != data.readInt());
                    com.liulishuo.filedownloader.model.FileDownloadHeader _arg7;
                    if ((0 != data.readInt())) {
                        _arg7 = com.liulishuo.filedownloader.model.FileDownloadHeader.CREATOR.createFromParcel(data);
                    } else {
                        _arg7 = null;
                    }
                    boolean _arg8;
                    _arg8 = (0 != data.readInt());
                    this.start(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_pause: {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    boolean _result = this.pause(_arg0);
                    reply.writeNoException();
                    reply.writeInt(((_result) ? (1) : (0)));
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements com.liulishuo.filedownloader.i.IFileDownloadIPCService {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void start(java.lang.String url, java.lang.String path, boolean pathAsDirectory, int callbackProgressTimes, int callbackProgressMinIntervalMillis, int autoRetryTimes, boolean forceReDownload, com.liulishuo.filedownloader.model.FileDownloadHeader header, boolean isWifiRequired) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(url);
                    _data.writeString(path);
                    _data.writeInt(((pathAsDirectory) ? (1) : (0)));
                    _data.writeInt(callbackProgressTimes);
                    _data.writeInt(callbackProgressMinIntervalMillis);
                    _data.writeInt(autoRetryTimes);
                    _data.writeInt(((forceReDownload) ? (1) : (0)));
                    if ((header != null)) {
                        _data.writeInt(1);
                        header.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    _data.writeInt(((isWifiRequired) ? (1) : (0)));
                    mRemote.transact(Stub.TRANSACTION_start, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public boolean pause(int downloadId) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(downloadId);
                    mRemote.transact(Stub.TRANSACTION_pause, _data, _reply, 0);
                    _reply.readException();
                    _result = (0 != _reply.readInt());
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_start = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
        static final int TRANSACTION_pause = (android.os.IBinder.FIRST_CALL_TRANSACTION + 4);
    }

    public void start(
        java.lang.String url, 
        java.lang.String path, 
        boolean pathAsDirectory, 
        int callbackProgressTimes, 
        int callbackProgressMinIntervalMillis,
        int autoRetryTimes, 
        boolean forceReDownload, 
        com.liulishuo.filedownloader.model.FileDownloadHeader header, 
        boolean isWifiRequired
    ) throws android.os.RemoteException;

    public boolean pause(int downloadId) throws android.os.RemoteException;
}
```
注意，生成的 java 文件内部，就有 IFileDownloadIPCCallback 接口的两个实现，它们分别是：

* `IFileDownloadIPCService.Stub` 就如注释所说，它是本地接口实现，这个本地在这个实际的例子中指的是下载进程，即下载进程里获得的 IFileDownloadIPCService 对象就是它。
* `IFileDownloadIPCService.Stub.Proxy` 如它的名称一样，它是用来代理远程的接口实现，这里的远程就是下载进程，即主进程拿到的 IFileDownloadIPCService 对象就是它。

## 使用 Binder 接口

FileDownloader 通过 `FileDownloadService` 服务开启下载进程，在该 Service 的 `onBind` 方法中，返回的是 `FDServiceSeparateHandler` 对象，该对象的定义如下：

```java
public class FDServiceSeparateHandler extends IFileDownloadIPCService.Stub {
}
```
即它实现的是 IFileDownloadIPCService.Stub 对象。而主进程如果获取属于它的 IFileDownloadIPCService 对象呢？FileDownloadService 本身是由主进程启动的，通过 `Context#bindService(Intent,ServiceConnection,int)` 方法启动并绑定下载服务。关键在于第二个参数 `ServiceConnection` ，`BaseFileServiceUIGuard` 是作为跨进程通信的入口，它同时实现了 `ServiceConnection` 接口：

```java
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    this.service = IFileDownloadIPCService.Stub.asInterface(service);
    ...
}
```

此时主进程便通过 `IFileDownloadIPCService.Stub` 的 `asInterface` 获取到了 IFileDownloadIPCService 对象。

# AIDL 原理

在使用介绍中，FileDownloader 主进程通过 IFileDownloadIPCService.Stub.asInterface(service) 方法拿到 IFileDownloadIPCService 对象。其中的参数 `service` 是一个 `IBinder` 对象。从该对象开始分析两个进程是如何拿到各自需要的 AIDL 接口对象。

## IBinder

该接口定义如下：

```java
/**
 * Base interface for a remotable object, the core part of a lightweight
 * remote procedure call mechanism designed for high performance when
 * performing in-process and cross-process calls.  This
 * interface describes the abstract protocol for interacting with a
 * remotable object.  Do not implement this interface directly, instead
 * extend from {@link Binder}.
 * 
 * <p>The key IBinder API is {@link #transact transact()} matched by
 * {@link Binder#onTransact Binder.onTransact()}.  These
 * methods allow you to send a call to an IBinder object and receive a
 * call coming in to a Binder object, respectively.  This transaction API
 * is synchronous, such that a call to {@link #transact transact()} does not
 * return until the target has returned from
 * {@link Binder#onTransact Binder.onTransact()}; this is the
 * expected behavior when calling an object that exists in the local
 * process, and the underlying inter-process communication (IPC) mechanism
 * ensures that these same semantics apply when going across processes.
 * 
 * <p>The data sent through transact() is a {@link Parcel}, a generic buffer
 * of data that also maintains some meta-data about its contents.  The meta
 * data is used to manage IBinder object references in the buffer, so that those
 * references can be maintained as the buffer moves across processes.  This
 * mechanism ensures that when an IBinder is written into a Parcel and sent to
 * another process, if that other process sends a reference to that same IBinder
 * back to the original process, then the original process will receive the
 * same IBinder object back.  These semantics allow IBinder/Binder objects to
 * be used as a unique identity (to serve as a token or for other purposes)
 * that can be managed across processes.
 * 
 * <p>The system maintains a pool of transaction threads in each process that
 * it runs in.  These threads are used to dispatch all
 * IPCs coming in from other processes.  For example, when an IPC is made from
 * process A to process B, the calling thread in A blocks in transact() as
 * it sends the transaction to process B.  The next available pool thread in
 * B receives the incoming transaction, calls Binder.onTransact() on the target
 * object, and replies with the result Parcel.  Upon receiving its result, the
 * thread in process A returns to allow its execution to continue.  In effect,
 * other processes appear to use as additional threads that you did not create
 * executing in your own process.
 * 
 * <p>The Binder system also supports recursion across processes.  For example
 * if process A performs a transaction to process B, and process B while
 * handling that transaction calls transact() on an IBinder that is implemented
 * in A, then the thread in A that is currently waiting for the original
 * transaction to finish will take care of calling Binder.onTransact() on the
 * object being called by B.  This ensures that the recursion semantics when
 * calling remote binder object are the same as when calling local objects.
 * 
 * <p>When working with remote objects, you often want to find out when they
 * are no longer valid.  There are three ways this can be determined:
 * <ul>
 * <li> The {@link #transact transact()} method will throw a
 * {@link RemoteException} exception if you try to call it on an IBinder
 * whose process no longer exists.
 * <li> The {@link #pingBinder()} method can be called, and will return false
 * if the remote process no longer exists.
 * <li> The {@link #linkToDeath linkToDeath()} method can be used to register
 * a {@link DeathRecipient} with the IBinder, which will be called when its
 * containing process goes away.
 * </ul>
 * 
 * @see Binder
 */
public interface IBinder {

    /**
     * Limit that should be placed on IPC sizes to keep them safely under the
     * transaction buffer limit.
     * @hide
     */
    public static final int MAX_IPC_SIZE = 64 * 1024;

    /**
     * Get the canonical name of the interface supported by this binder.
     */
    public @Nullable String getInterfaceDescriptor() throws RemoteException;

    public boolean pingBinder();

    public boolean isBinderAlive();
    
    /**
     * Attempt to retrieve a local implementation of an interface
     * for this Binder object.  If null is returned, you will need
     * to instantiate a proxy class to marshall calls through
     * the transact() method.
     */
    public @Nullable IInterface queryLocalInterface(@NonNull String descriptor);

    public interface DeathRecipient {
        public void binderDied();
    }

    public void linkToDeath(@NonNull DeathRecipient recipient, int flags)
            throws RemoteException;

    public boolean unlinkToDeath(@NonNull DeathRecipient recipient, int flags);
}
```

首先从它的注释可以了解到该接口用来描述远过程调用，它主要有以下接口：

* `transact/onTransact` 通过该方法发送给远程 IBinder 对象发送请求，并将数据交由 `onTransact` 方法消费，参数信息以及消费结果都是通过 `Parcel` 对象传递。
* `pingBinder` 可以用来检查远程对象是否还存活着。
* `linkToDeath` 可以注册一个 `DeathRecipient` 对象监听远程进程是否死亡。
* `queryLocalInterface` 这个就是为了给本地进程获取 AIDL 接口。

除此之外，每个进程内部维护了一个线程池进行 transaction 操作，不过对于 `transact` 接口本身来说，它是一个同步方法，一定会等到 `onTransact` 完成后才返回。

## 实例总结

基于现有信息，回顾我们之前的例子，可以更加清楚主进程和下载进程分别获取 IFileDownloadIPCService 接口的过程。

### 主进程

回顾 `IFileDownloadIPCService.Stub.asInterface(service)` 方法：

```java
/**
 * Cast an IBinder object into an com.liulishuo.filedownloader.i.IFileDownloadIPCService interface,
 * generating a proxy if needed.
*/
public static com.liulishuo.filedownloader.i.IFileDownloadIPCService asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof com.liulishuo.filedownloader.i.IFileDownloadIPCService))) {
        return ((com.liulishuo.filedownloader.i.IFileDownloadIPCService) iin);
    }
    return new com.liulishuo.filedownloader.i.IFileDownloadIPCService.Stub.Proxy(obj);
}
```

首先调用 IBinder 对象的 `queryLocalInterface` 方法，主进程传递过来的 IBinder 对象调用该接口只会返回 null（原因暂时不在这里分析），所以最终它返回的是 `IFileDownloadIPCService.Stub.Proxy` 对象。作为 IFileDownloadIPCService 接口的调用方，我们以 `pause` 方法为例观察一下：

```java
@Override
public boolean pause(int downloadId) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    boolean _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        _data.writeInt(downloadId);
        mRemote.transact(Stub.TRANSACTION_pause, _data, _reply, 0);
        _reply.readException();
        _result = (0 != _reply.readInt());
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

构造好参数后，调用了远程 IBinder 对象的 transact 方法，前面说过这个远程 IBinder 对象，即下载进程的 IFileDownloadIPCService 接口，是 `IFileDownloadIPCService.Stub` 对象。来看看它的 `onTransact` 方法：

```java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    java.lang.String descriptor = DESCRIPTOR;
    switch (code) {
        ...
        case TRANSACTION_pause: {
            data.enforceInterface(descriptor);
            int _arg0;
            _arg0 = data.readInt();
            boolean _result = this.pause(_arg0);
            reply.writeNoException();
            reply.writeInt(((_result) ? (1) : (0)));
            return true;
        }
        default: {
            return super.onTransact(code, data, reply, flags);
        }
    }
}
```

通过检查 Parcel 中的元信息，可以响应主进程的 `pause` 调用，最终调用下载进程中开发者自己实现的 `pause` 接口。

### 下载进程

注意 `IFileDownloadIPCService.Stub` 对象实现的是 `android.os.Binder` 接口，它也是 `IBinder` 接口的默认实现。在 `IFileDownloadIPCService.Stub` 的构造函数里：

```java
public Stub() {
    this.attachInterface(this, DESCRIPTOR);
}
```

`Binder` 中 `attachInterface` 的定义如下：

```java
/**
 * Convenience method for associating a specific interface with the Binder.
 * After calling, queryLocalInterface() will be implemented for you
 * to return the given owner IInterface when the corresponding
 * descriptor is requested.
*/
public void attachInterface(@Nullable IInterface owner, @Nullable String descriptor) {
    mOwner = owner;
    mDescriptor = descriptor;
}
```

再对照它的 `queryLocalInterface` 方法：

```java
public @Nullable IInterface queryLocalInterface(@NonNull String descriptor) {
    if (mDescriptor != null && mDescriptor.equals(descriptor)) {
        return mOwner;
    }
    return null;
}
```

所以说，如果下载进程中的 `asInterface` 方法，返回的依然是 `IFileDownloadIPCService.Stub` 对象。

# 参考
> https://developer.android.com/guide/components/aidl?hl=zh-cn

> http://gityuan.com/2015/10/31/binder-prepare/