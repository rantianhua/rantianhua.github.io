---
layout: post
title:  "MediaCodec 使用和源码分析"
date:   2018-02-09 13:09:00
categories: Android
---

# 前言

项目中经常需要进行视频的播放，播放首先需要进行解码操作。在 Android 中，将原始编码后的视频数据解码，再将解码数据提供给底层的 SurfaceFlinger 进行最后的渲染，是通过 MediaCodec 来完成。本片文章的重点将放在 MediaCodec 的创建以及配置的过程（源码部分基于 `android-8.0.0_r1`）。

# 基本使用

## 视频解封装

视频进程传播时往往被封装成约定的格式，比如常见的 MP4 格式。第一步就要解封装获取相应的 `MediaFormat` 信息。

```kotlin
private fun extactMediaFormat(filePath: String): MediaFormat? {
    val extractor = MediaExtractor()
    extractor.setDataSource(filePath)
    val count = extractor.trackCount
    for (i in 0 until count) {
        val format = extractor.getTrackFormat(i)
        if (format.getString(MediaFormat.KEY_MIME).startsWith("video")) {
            return format
        }
    }
    return null
}
```

MediaFormat 封装了描述媒体资源的所有信息，比如视频的宽高、帧率、MIME 等等。

## 创建解码器

```kotlin
private fun createMediaCodec(format: MediaFormat, surface: Surface): MediaCodec {
    val codecList = MediaCodecList(MediaCodecList.REGULAR_CODECS)
    val codecName = codecList.findDecoderForFormat(format)
    val codec = MediaCodec.createByCodecName(codecName)
    codec.configure(format, surface, null, 0)
    return codec
}
```

通过 `MediaCodecList` 可获得系统所有的编解码器，再通过它提供的 `findDecoderForFormat` 可以获取该 MediaFormat 适合的解码器名称，然后根据解码器名称创建 `MediaCodec` 对象。创建完成之后一定要进行 config 操作。

## 使用解码器

### 同步使用

```kotlin
private fun feedInputBuffer(mediaCodec: MediaCodec, extractor: MediaExtractor) {
    val index = mediaCodec.dequeueInputBuffer(0)
    val buffer = mediaCodec.getInputBuffer(index) ?: return
    val sampleSize = extractor.readSampleData(buffer, 0)
    if (sampleSize < 0) {
        mediaCodec.queueInputBuffer(index, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM)
    } else {
        mediaCodec.queueInputBuffer(index, 0, sampleSize, extractor.sampleTime, 0)
        extractor.advance()
    }
}

private fun readOutputBuffer(mediaCodec: MediaCodec) {
    val info = MediaCodec.BufferInfo()
    val index = mediaCodec.dequeueOutputBuffer(info, 0)
    val buff = mediaCodec.getOutputBuffer(index)
    ...
    mediaCodec.releaseOutputBuffer(index, true)
}
```

`feedInputBuffer` 演示如何往 InputBuffer 填充数据，即给 MediaCodec 喂数据；`readOutputBuffer` 演示如何从 MediaCodec 中读取解码后的数据。这两个方法通常放在两个独立的线程中进行循环操作。最终整个过程就是 `dequeueInputBuffer` 获取缓冲区 -> `queueInputBuffer` 填充缓冲区 -> `dequeueOutputBuffer` 获取解码帧数据 -> `releaseOutputBuffer` 显示画面并释放缓冲区。

### 异步使用

```kotlin
private fun attachMediaCodecCallback(mediaCodec: MediaCodec) {
    mediaCodec.setCallback(object : MediaCodec.Callback() {
        override fun onOutputBufferAvailable(codec: MediaCodec, index: Int, info: MediaCodec.BufferInfo) {
        }

        override fun onInputBufferAvailable(codec: MediaCodec, index: Int) {
        }

        override fun onOutputFormatChanged(codec: MediaCodec, format: MediaFormat) {
        }

        override fun onError(codec: MediaCodec, e: MediaCodec.CodecException) {
        }
    })
}
```

`MediaCodec.Callback` 是一个抽象类，将整个过程描述为具体的接口，然后使用方实现接口进行具体操作即可。

# 工作原理

![工作原理图示]({{site.baseurl}}/images/codec_process.png)

MediaCodec 的整个工作方式如上图所示，整个过程都是在操作输入和输出缓冲区。客户端先获取一个空闲的 input buffer ，然后填入数据。填满数据的 input buffer 交还给 codec 进行编解码处理，处理完成后 codec 会将数据放入一个 output buffer 中。最后客户端请求 output buffer ，消耗其处理过的数据后释放该 buffer 。

# 源码分析

## Surface 简介

前面在配置 MediaCodec 的时候需要提供一个 Surface 对象。对于视频数据，最终是要被渲染到屏幕上的。渲染操作实际上是操作的 Surface 对象，在 Java 层，可以理解为绘制的画布，在 Native 层，就是一个 raw buffer ，最终该 buffer 的数据会被交给 SurfaceFlinger 进程合成显示到屏幕上。

## MediaCodec 的创建

### MediaCodec.createByCodecName

前面使用该方法进行创建操作，最终会调用到下面的方法：

```java
private MediaCodec(@NonNull String name, boolean nameIsType, boolean encoder) {
    ...
    native_setup(name, nameIsType, encoder);
}
```

其中 `nameIsType` 和 `encoder` 都是 false 。`native_setup` 将会通过 jni 调用 native 层代码，具体在源码文件为 [android_media_MediaCodec.cpp](https://cs.android.com/android/platform/superproject/+/android-8.0.0_r1:frameworks/base/media/jni/android_media_MediaCodec.cpp) 。

### android_media_MediaCodec_native_setup

通过传入的参数实例化一个 `JMediaCodec` 结构体，然后调用其 `initCheck` 方法检查 native 层的 MediaCodec 是否成功创建并实例化。如果提供的编解码器名字没有或者没有足够内存的情况下，都会抛出 `IllegalArgumentException` 。如果是其他错误，则会抛出 `IOException` 。

### JMediaCodec

该结构体定义在 `android_media_MediaCodec.cpp` 中，继承自 `AHandler` ，它其实是 native 层对应于 Java 层的 `MediaCodec` 对象。从 Java 层对 MediaCodec 的操作到 native 层都是通过它来进行委托的。

构造函数中创建了新的 `ALooper` 对象并开启 Looper 。然后调用 [MediaCodec](https://cs.android.com/android/platform/superproject/+/android-8.0.0_r1:frameworks/av/media/libstagefright/MediaCodec.cpp) 的 `CreateByComponentName` 方法创建 native 层的 codec 对象。

### MediaCodec::CreateByComponentName

创建 native 层的 MediaCodec 对象，其构造函数中的 `ALooper` 来自前面 `JMediaCodec`。 

创建完成后立马调用 `codec->init(name, false /* nameIsType */, false /* encoder */);` 方法。

### MediaCodec::init

通过 `GetCodecBase(name, nameIsType)` 返回具体的编/解码器 `mCodec` ，其类型为 `CodecBase` 。然后还要通过 `MediaCodecList::findCodecByName` 和 `MediaCodecList::getCodecInfo` 来判断该编/解码器是否是为 video 工作的。如果是，则新建一个 `ALooper` 并将其命名为 `mCodecLooper` ，此时 MediaCodec 将会有两个 ALooper ，另一个名为 `mLooper` ，来自于 `JMediaCodec` 。

此 MediaCodec 也是一个 `AHandler` 对象，通过 `mLooper->registerHandler(this);` 让其相应 `mLooper` 维护的消息队列。

然后是对 `mCodec` 以及其 buffer channel 设置 Callback ，该 callback 的形式都是当相应事件发生时发送一个 what 值为 `kWhatCodecNotify` 的 AMessage。最后是示例化一个 `AMessage` 对象，其 what 值为 `kWhatInit` ，该消息会被 `MediaCodec::onMessageReceived` 处理。

### MediaCodec::GetCodecBase

如果 codec 名称以 `omx.` 开头（大小写忽略），返回一个 `ACodec` 对象；如果以 `android.filter.` 开头，返回一个 `MediaFilter` 对象。

### MediaCodec::onMessageReceived~kWhatInit

检查当前装填，如果不为 `UNINITIALIZED` 则返回 `INVALID_OPERATION` 。然后将当前状态设置为 `INITIALIZING` 。然后进行 `mCodec->initiateAllocateComponent(format);` 初始化操作，完成后会会回调之前设置的 Callback ，即响应 what 值为 `kWhatCodecNotify` 的 AMessage 。

### MediaCodec::onMessageReceived~kWhatCodecNotify

此时 AMessage 中还存储了一个 what 值，由 `mCodec->initiateAllocateComponent` 设置，其取值范围如下：

- `kWhatError` 此时输出具体的错误信息，然后检查当前 MediaCodec 状态，此时是 `INITIALIZING` ，那么重新将状态设置为 `UNINITIALIZED` ，最后抛出异常。其他状态情况这里不错细致记录。
- `kWhatComponentAllocated` 此时代表具体的编/解码器初始化成功，然后获取该 codec 的 component name ，如果是 `OMX.google.` 开头，说明是软编/解码器，添加 `kFlagUsesSoftwareRenderer` 这个 flag 。

到这里，创建工作算是完成。

## MediaCodec 的配置

### MediaCodec.configure

以 key-value 的方式分别读取 MediaFormat 中的信息，所有的 key 放入 `keys` 这个字符串数组，所有的 value 放入 `values` 这个数组。最终调用 `native_configure(keys, values, surface, crypto, descramblerBinder, flags);`

### android_media_MediaCodec_native_config

将所有的 key/value 数据记录到一个 AMessage 中。然后调用 [android_view_Surface.cpp](https://cs.android.com/android/platform/superproject/+/android-8.0.0_r1:frameworks/base/core/jni/android_view_Surface.cpp) 的 `android_view_Surface_getSurface` 方法将 Java 层的 Surface 对象转换为 native 层的 [Surface](https://cs.android.com/android/platform/superproject/+/android-8.0.0_r1:frameworks/native/libs/gui/Surface.cpp) 对象。再获取该 Surface 对象的 [IGraphicBufferProducer](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/gui/include/gui/IGraphicBufferProducer.h) 对象，它为 MediaCodec 提供图形缓冲区。

利用上面获取的参数，调用 `codec->configure(format, bufferProducer, crypto, descrambler, flags);`

### JMediaCodec::configure

利用 `bufferProducer` 参数创建新的 native 层 Surface 对象`mSurfaceTextureClient` 。然后调用 `mCodec->configure(
            format, mSurfaceTextureClient, crypto, descrambler, flags);` 。

### MediaCodec::configure

此方法完整定义为：

```C++
status_t MediaCodec::configure(
        const sp<AMessage> &format,
        const sp<Surface> &surface,
        const sp<ICrypto> &crypto,
        const sp<IDescrambler> &descrambler,
        uint32_t flags) {
}
```

先实例化一个 AMessage ，其 what 值为 `kWhatConfigure` ，如果 `mIsVideo` 为 true 的话，会根据 video 的 width 和 height 检查编码器缓冲区是否过大，检查条件为：

```C++
if (mInitIsEncoder
        && (uint64_t)mVideoWidth * mVideoHeight > (uint64_t)INT32_MAX / 4) {
    ALOGE("buffer size is too big, width=%d, height=%d", mVideoWidth, mVideoHeight);
    return BAD_VALUE;
}
```

将 format 信息添加到 AMessage ，将 Surface 对象也记录到 AMessage 并将该 message 复制给 `mConfigureMsg` 进行保存，后面应该需要用到。

最后发送 message 作进一步处理。

### MediaCodec::onMessageReceived~kWhatConfigure

先检查如果状态不是 `INITIALIZED` 则返回 `INVALID_OPERATION` 。从 message 中取出 Surface 和 format 。

判断 Surface 对象不为空，那么将其设置给 format 这个 AMessage ，其 key 为 `native-window` 。然后调用 `handleSetSurface` 方法。

将状态设置为 `CONFIGURING` ，调用 `mCodec->initiateConfigureComponent(format);` 进行具体的 codec 的配置操作。

### MediaCodec::handleSetSurface

如果当前 `mSurface` 对象不为空，说明有旧的 surface ，那么断开与旧 surface 的连接。如果参数中的 `surface` 不为空，利用这个 surface 建立新的连接，成功后，将其赋值给 `mSurface` 。

### MediaCodec::onMessageReceived~kWhatCodecNotify & kWhatComponentConfigured

和前面创建流程一样，这里不重复说明。先检查状态：

```C++
if (mState == UNINITIALIZED || mState == INITIALIZED) {
    // In case a kWhatError message came in and replied with error,
    // we log a warning and ignore.
    ALOGW("configure interrupted by error, current state %d", mState);
    break;
}
CHECK_EQ(mState, CONFIGURING);
```

如果状态为 `UNINITIALIZED` 或者 `INITIALIZED` ，不继续处理。然后再要求装状态必须为 `CONFIGURING` 。将 `mHaveInputSurface` 置为 false 便于之后重新设置 input surface 。然后获取 `mInputFormat` 和 `mOutputFormat` 。最后设置状态为 `CONFIGURED`，配置过程完成 。

# 参考 

> https://david1840.github.io/2019/01/08/Android%E9%9F%B3%E8%A7%86%E9%A2%91-%E4%B8%89-MediaCodec%E7%A1%AC%E7%BC%96%E7%A1%AC%E8%A7%A3/

> https://cs.android.com/android/platform/superproject/+/android-8.0.0_r1:frameworks/av/media