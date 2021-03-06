---
layout: post
title:  "Binder Native 层实现原理"
date:   2018-01-07 21:00:00
categories: Android
---

# IPC 机制

Binder 是 Android 在 Linux 基础上特有的一种 IPC 机制。IPC 面向的对象是进程，进程是操作系统资源管理的最小单位。在日常的 Android 开发中，对进程最直观的理解就是一个应用程序，当然，他们之间并不是一对一的关系，因为一个应用程序也可以由多个进程组成。进程之间的资源是完全隔离的，他们之间的通信不像 A 持有 B ，A 就可以访问 B 这么简单。下面这张图非常简单直观地描述了 IPC 的过程：

![IPC 原理]({{site.baseurl}}/images/what's_ipc.png)

因为资源隔离的关系，进程之间无法直接通信，需要通过内核空间中转。在此概念的基础上，进一步去理解 Binder 的实现原理。

# Binder 原理

再次寄出 Gityuan 画的一张非常好的图：

![Binder 原理]({{site.baseurl}}/images/binder_process.png)

整个通信采用 C/S 架构，核心在于 Native 层的 Service Manager 和 Binder 驱动。前者是所有 Binder 通信对象的服务端，类似 Binder 的管家服务。后者代表 IPC 原理图中的内核空间，Client 和 Server 都是无法和 ServiceManager 直接通信，需要通过这一层中转。

## 注册服务

先来看看图中的第一步，将基于 `android-9.0.0_r1` 源码了解 Media 服务如何注册到 ServiceManager 的。

Media 服务的入口在 `main_mediaserver.cpp` 文件：

```C++
int main(int argc __unused, char **argv __unused)
{
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());
    ...
    MediaPlayerService::instantiate();
    ...
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

先是获取 `ProcessState` 对象，`self` 函数声明如下：

```C++
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}
```

ProcessState 是一个单例对象，构造时传入 binder 驱动的名称，看看构造函数里做了什么：

```C++
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}
```

在实例化的时候便调用了 `open_driver` 方法打开 binder 驱动，并用 `mDriverFD` 记录驱动 id 。也就是说，Media 服务在一开始便通过获取 ProcessState 这个单例对象打开了 Binder 驱动，为后续和 Binder 进行交互做好了准备。

然后通过 `defaultServiceManager()` 获取的 `IServiceManager` 接口用于和 ServiceManager 进程通信，其实例类为 `BpServiceManager`。

接下来是最重要的注册过程，也就是上面的 `MediaPlayerService` 的 `instantiate()` 方法：

```C++
void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```

直接看 BpServiceManager 的 `addService` 实现：

```C++
virtual status_t addService(const String16& name, const sp<IBinder>& service,
                            bool allowIsolated, int dumpsysPriority) {
    Parcel data, reply;
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    data.writeString16(name);
    data.writeStrongBinder(service);
    data.writeInt32(allowIsolated ? 1 : 0);
    data.writeInt32(dumpsysPriority);
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```

其中 `name` 参数就是 `media.player` 。这里利用 Parcel 对象写入参数，然后通过 `remote` 调用 `transact` 方法，和 Java 层利用 AIDL 进行跨进程调用如出一辙。记住这里 `transact` 第一个参数是 `ADD_SERVICE_TRANSACTION` ，表示像 ServiceManager 服务注册这里的 Media 服务。

`remote()` 指向 `BpBinder` 对象，看看它的 `transact` 方法：

```C++
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        ...
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        ...
    }
    ...
}
```

继续到 `IPCThreadState` 的 `transact` 方法：

```C++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    ...
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        ...
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        ...
    } else {
        err = waitForResponse(nullptr, nullptr);
    }

    return err;
}
```
在 `IPCThreadState` 中有两个 Parcel 对象 `mIn` 和 `mOut`，前者用来接收远程写回的结果，后者用来写入本地调用的参数数据。`writeTransactionData` 正是把 `BC_TRANSACTION` 等数据写入到 `mOut` 对象。然后到 `waitForResponse` 方法：

```C++
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        ...
        cmd = (uint32_t)mIn.readInt32();
        ...
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            ...
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```

在 while 循环中，每次先执行 `talkWithDriver` 然后判断 Binder 驱动通过 `mIn` 写回的结果。

```C++
status_t IPCThreadState::talkWithDriver(bool doReceive)
{

    ...
    status_t err;
    do {
        ...
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
        ...
    } while (err == -EINTR);
    ...
    return err;
}
```

这里的有频繁读写数据的过程，但最最重要的是 `ioctl` 调用，`mDriverFD` 是 ProcessState 对象在实例化就获取了的，这也是 Media 进程 Binder 驱动真正进行交互的过程。交互的结果通过 `mIn` 这个 Parcel 对象写回，然后 `waitForResponse` 里面循环读取结果直到添加服务的任务结束。

在 Binder 驱动内部再到 ServiceManager 的调用过程留作下次分析。

## 获取服务

依然以获取 Media 服务为例。入口为 `frameworks/av/media/libmedia/IMediaDeathNotifier.cpp` 的 `getMediaPlayerService` 方法：

```C++
IMediaDeathNotifier::getMediaPlayerService()
{
    ALOGV("getMediaPlayerService");
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService == 0) {
        sp<IServiceManager> sm = defaultServiceManager();
        sp<IBinder> binder;
        do {
            binder = sm->getService(String16("media.player"));
            if (binder != 0) {
                break;
            }
            ALOGW("Media player service not published, waiting...");
            usleep(500000); // 0.5 s
        } while (true);

        if (sDeathNotifier == NULL) {
            sDeathNotifier = new DeathNotifier();
        }
        binder->linkToDeath(sDeathNotifier);
        sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    ALOGE_IF(sMediaPlayerService == 0, "no media player service!?");
    return sMediaPlayerService;
}
```

通过 `BpServiceManager` 的 `getService` 方法去获取 Media 服务，如果服务还未注册上，则等 0.5s 再试，知道获取到为止。那么重点来看 `BpServiceManager#getService` 方法：

```C++
virtual sp<IBinder> getService(const String16& name) const
{
    ...
    const long timeout = uptimeMillis() + 5000;
    ...
    int n = 0;
    while (uptimeMillis() < timeout) {
        n++;
        ...
        sp<IBinder> svc = checkService(name);
        if (svc != NULL) return svc;
    }
    ALOGW("Service %s didn't start. Returning NULL", String8(name).string());
    return NULL;
}
```

隐藏了部分等待系统启动的逻辑一些休眠等待的逻辑，核心就是在 5s 内间歇性地通过 `checkService` 方法获取服务，直到成功或者超时，其中 `name` 参数就和添加服务时一样的 `media.player` 。

```C++
virtual sp<IBinder> checkService( const String16& name) const
{
    Parcel data, reply;
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    data.writeString16(name);
    remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
    return reply.readStrongBinder();
}
```

依然调用到 `remote()` 返回的 `BpBinder` 的 `transact` 方法，和添加服务时类似，不过此时 `code` 参数为 `CHECK_SERVICE_TRANSACTION`。接下来的调用流程和添加服务时一致，都是通过 `IPCThreadState` 对象最终和 Binder 驱动进行 Binder 通信，最终通过 `reply.readStrongBinder()` 返回 Media 服务。

`frameworks/native/libs/binder/Parcel.cpp` :

```C++
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    // Note that a lot of code in Android reads binders by hand with this
    // method, and that code has historically been ok with getting nullptr
    // back (while ignoring error codes).
    readNullableStrongBinder(&val);
    return val;
}

status_t Parcel::readNullableStrongBinder(sp<IBinder>* val) const
{
    return unflatten_binder(ProcessState::self(), *this, val);
}
```

最终走到 `unflatten_binder` ，其第一个参数是每个进程唯一的 `ProcessState` 对象，第二个参数指向这个 Parcel 对象本身，第三个是需要写入结果的目标对象。

```C++
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);

    if (flat) {
        switch (flat->hdr.type) {
            case BINDER_TYPE_BINDER:
                ...
            case BINDER_TYPE_HANDLE:
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```

`flat_binder_object` 结构体定义如下：

```C++
typedef __u32 binder_uintptr_t;
struct binder_object_header {
  __u32 type;
};
struct flat_binder_object {
  struct binder_object_header hdr;
  __u32 flags;
  union {
    binder_uintptr_t binder;
    __u32 handle;
  };
  binder_uintptr_t cookie;
};
```

它代表了在进程之间传递的 Binder 对象。当 `flbinder_object_header.type` 为 `BINDER_TYPE_BINDER` 时，代表 Server 进程向 ServiceManager 注册服务；当其为 `BINDER_TYPE_HANDLE` 时表示 Client 向 Server 请求代理。这里就是调用进程向 ServiceManager 进程请求 Media 服务的代理。然后看看 `ProcessState#getStrongProxyForHandle` 返回结果：

```C++
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;
    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.
                //
                // TODO: add a driver API to wait for context manager, or
                // stop special casing handle 0 for context manager and add
                // a driver API to get a handle to the context manager with
                // proper reference counting.

                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }

            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

`lookupHandleLocked` 解析 `flat_binder_object` 结构体数据。如果通过本次解析的结果里面能获取 Media 服务的 Binder 代理，那么直接返回。否则创建一个新的 BpBinder 对象并返回。有个异常 case 是在 ServiceManager 服务还没有启动或者挂掉的情况下，会通过 `PING_TRANSACTION` 这个 IPC 调用检查 ServiceManager 状态，如果已经死掉了，那么本次获取将会失败。

前面说到 Binder 驱动中使用 `flat_binder_object` 这个结构体传递 Binder 对象，那么驱动具体如何使用该对象？各服务的 Binder 对象细节又是如何处理的？这部分 Binder 驱动内部的细节留作下次学习。

# 总结

通过对 Media 服务的注册和获取，了解了 Binder 在 native 层是如何运作的。Binder 是 Android 独有的，而 Linux 本身也有进程间通信的方式：

- __管道__ 在创建时分配一小块内存来通信，缓冲区大小有限，只能承载无格式字节流。
- __消息队列__ 在内核中维护一个消息链表，进程根据被赋予的权限进行写队列和读队列的操作。
- __信号量__ 信号量是一个计数器，实际是一种锁机制，防止同一时刻多个进程访问资源。
- __信号__ 用于通知进程某个时间发生了，本身不适合用来传递数据。
- __共享内存__ 映射一段能被其他进程直接访问的内存，这是最快的 IPC 通信方式。
- __套接字__ Socket 通信是更为一般、更广范围的跨进程通信，进程可以运行在不同的机器上。

这些已有的通信方法，都有一些本身的缺陷。管道缓冲区太小，对数据的限制多；效率队列需要将信息复制两次；共享内存虽然效率极高，但需要额外处理同步问题；套接字这种通用接口通常用来不同机器的跨网络传输，对同一机器内的进程来说，效率实在是低了些；信号量只是一种同步手段，不是设计来传递数据的；信号也不适合传递数据，通常用来中断控制。

针对这些问题，Binder 机制有做了哪些改进呢？

1. 性能方面，Binder 利用 `mmap` 这个系统调用在内核空间和接收方用户空间的数据缓存区之间做了一次内存映射，相当于直接把数据拷贝到了接收方的用户空间，所以它的数据只拷贝了一次。
2. 稳定性方面，Binder 机制采用 C/S 架构，有了客户端和服务端之分，各端职责清晰，有更好的稳定性。
3. 安全方面，以上 Linux 的进程通信方式无法鉴别身份，Binder 进制中，各个服务都需要进行注册，每个进程都有自己的 UID 标识，Server 端可以通过 UID/PID 鉴别身份，进行权限控制。

# 参考

> http://gityuan.com/2015/10/31/binder-prepare/

> https://blog.csdn.net/gatieme/article/details/50908749

> https://www.zhihu.com/question/39440766