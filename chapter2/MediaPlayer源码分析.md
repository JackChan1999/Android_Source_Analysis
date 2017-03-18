# Media Player 源码分析

**目录**
- [1.简介](#简介)
- [2.Media Server](#2media-server)
- [3.MediaPlayer 调用流程](#3mediaplayer-调用流程)
    - [3.1 构造函数](#31-构造函数)
    - [3.2 设置数据源](#32-设置数据源)
        - [3.2.1 获取 MediaPlayerService 接口](#321-获取-mediaplayerservice-接口)
        - [3.2.2 获取 MediaPlayer 接口](#322-获取-mediaplayer-接口)
        - [3.2.3 设置数据源](#323-设置数据源)
    - [3.3 Prepare](#33-prepare)
    - [3.4 start](#34-start)
    - [3.5 pause](#35-pause)
    - [3.6 stop](#36-stop)
    - [3.7 release](#37-release)
- [4.总结](#4总结)
- [5.参考](#5参考)



## 1.简介
MediaPlayer 中大部分的功能使用 C++ 实现，Java 这边做的工作大部分是 JNI 的调用，这篇文章主要分析了常用的几个接口对应 C++ 实现和 Media Server。 


## 2.Media Server

Media Server 整体的架构是 C/S 架构，C 和 S 之间的通讯是 IPC，具体来说是 Binder。Media Server 中大量的用到了 Binder。整个架构将播放控制、视频、音频、相机等和多媒体有关的这些包装成不同的服务，通过 IPC 解耦。下图是 Google 关于 Android 中 [Media 引擎](https://source.android.com/devices/media.html)的架构做的关系图。

![Media 架构](https://source.android.com/devices/images/ape_fwk_media.png)

在系统 mediaserver 作为一个单独的进程，负责整个系统的音频视频编解码工作：

![media server 进程](http://7xisp0.com1.z0.glb.clouddn.com/media_server_process.png)

下面先从 Java 层的调用顺序开始看一下整个调用的流程。

## 3.MediaPlayer 调用流程

[MediaPlayer](https://developer.android.com/reference/android/media/MediaPlayer.html) 中涉及到的主要函数都是通过 JNI 来完成的，MediaPlayer.java 对应的是 [android_media_MediaPlayer.cpp](https://android.googlesource.com/platform/frameworks/base/+/android-5.1.1_r18/media/jni/android_media_MediaPlayer.cpp)。其中的对应关系如下，省略了一部分不是特别重要的。

```cpp

static JNINativeMethod gMethods[] = {
    {
        "nativeSetDataSource",
        "(Landroid/os/IBinder;Ljava/lang/String;[Ljava/lang/String;"
        "[Ljava/lang/String;)V",
        (void *)android_media_MediaPlayer_setDataSourceAndHeaders
    },
    {"_setDataSource",       "(Ljava/io/FileDescriptor;JJ)V",    (void *)android_media_MediaPlayer_setDataSourceFD},
    {"_prepare",            "()V",                              (void *)android_media_MediaPlayer_prepare},
    {"prepareAsync",        "()V",                              (void *)android_media_MediaPlayer_prepareAsync},
    {"_start",              "()V",                              (void *)android_media_MediaPlayer_start},
    {"_stop",               "()V",                              (void *)android_media_MediaPlayer_stop},
    {"getVideoWidth",       "()I",                              (void *)android_media_MediaPlayer_getVideoWidth},
    {"getVideoHeight",      "()I",                              (void *)android_media_MediaPlayer_getVideoHeight},
    {"seekTo",              "(I)V",                             (void *)android_media_MediaPlayer_seekTo},
    {"_pause",              "()V",                              (void *)android_media_MediaPlayer_pause},
    {"isPlaying",           "()Z",                              (void *)android_media_MediaPlayer_isPlaying},
    {"getCurrentPosition",  "()I",                              (void *)android_media_MediaPlayer_getCurrentPosition},
    {"getDuration",         "()I",                              (void *)android_media_MediaPlayer_getDuration},
    {"_release",            "()V",                              (void *)android_media_MediaPlayer_release},
    {"_reset",              "()V",                              (void *)android_media_MediaPlayer_reset},
    {"native_init",         "()V",                              (void *)android_media_MediaPlayer_native_init},
    {"native_setup",        "(Ljava/lang/Object;)V",            (void *)android_media_MediaPlayer_native_setup},
};

```

Java 这边的调用顺序通常是：

```java
MediaPlayer mp = new MediaPlayer();
mp.setDataSource("/sdcard/test.mp3");
mp.prepare();
mp.start();
mp.pause();
mp.stop();
mp.release();
```

下面将按照这个顺序一步一步来分析。

### 3.1 构造函数

在 MediaPlayer 的构造函数中调用了 native 的 android_media_MediaPlayer_native_setup 方法:

```java

public MediaPlayer() {
    ...

    /* Native setup requires a weak reference to our object.
     * It's easier to create it here than in C++.
     */
    native_setup(new WeakReference<MediaPlayer>(this));
}

```

setup 方法中创建了 MediaPlayer，同时也设置了回调函数。其中最后一行的 setMediaPlayer 将 MediaPlayer 的指针保存成一个 Java 对象，之后可以看到 getMediaPlayer 通过同样的方法获取到该对象的指针。

```cpp
static void
android_media_MediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)

{
    ALOGV("native_setup");
    sp<MediaPlayer> mp = new MediaPlayer();
    if (mp == NULL) {
        jniThrowException(env, "java/lang/RuntimeException", "Out of memory");
        return;
    }
    // create new listener and give it to MediaPlayer
    sp<JNIMediaPlayerListener> listener = new JNIMediaPlayerListener(env, thiz, weak_this);
    mp->setListener(listener);
    // Stow our new C++ MediaPlayer in an opaque field in the Java object.
    setMediaPlayer(env, thiz, mp);

}

static sp<MediaPlayer> setMediaPlayer(JNIEnv* env, jobject thiz, const sp<MediaPlayer>& player)
{
    Mutex::Autolock l(sLock);
    /* 由于指针的大小和 long 的大小是一样的，所以这里的 get 和 set 可以将
       MediaPlayer 的指针取出或存入*/
    sp<MediaPlayer> old = (MediaPlayer*)env->GetLongField(thiz, fields.context);
    // 判断 mediaplayer 是否为空指针，如果不是，这 sp 引用计数加1
    if (player.get()) {
        player->incStrong((void*)setMediaPlayer);
    }
    // 如果之前存过 mediaplayer，检查之前存的是否为空，如果不为空，则引用计数减1
    if (old != 0) {
        old->decStrong((void*)setMediaPlayer);
    }
    // 将新的 mediaplayer 指针存入 Java 对象中
    env->SetLongField(thiz, fields.context, (jlong)player.get());
    return old;
}
```

这里解释两点：

**1**.由于指针的大小和 long 的大小是一样的，所以可以通过 SetLongField 和 GetLongField 来保存函数指针。

**2**.由于 shared_ptr 和 weak_ptr 是 C++11 中才加入的，所以源码中实现了 sp 和 wp 作为智能指针来使用，这里可以看到代码中手动控制了引用计数。

现在一个 mediaplayer 对象就创建好了，其他的 jni 中对应的方法几乎都是调用 mediaplayer 来完成的。

### 3.2 设置数据源

[setDataSource](https://developer.android.com/reference/android/media/MediaPlayer.html#setDataSource%28java.io.FileDescriptor%29) 对应的是 jni 中的这个函数：

```cpp
static void android_media_MediaPlayer_setDataSourceFD(JNIEnv *env, jobject thiz, jobject fileDescriptor, jlong offset, jlong length)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    // mediaplayer 为空
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }
    // 文件描述符为空
    if (fileDescriptor == NULL) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return;
    }
    int fd = jniGetFDFromFileDescriptor(env, fileDescriptor);
    ALOGV("setDataSourceFD: fd %d", fd);
    // 调用 mp->setDataSource(fd, offset, length) 为 mediaplayer 设置数据源
    process_media_player_call( env, thiz, mp->setDataSource(fd, offset, length), "java/io/IOException", "setDataSourceFD failed." );
}

```

可以看到函数开始先做了空指针的判断，之后调用了 mp->setDataSource(fd, offset, length) 方法来给 mediaplayer 设置数据源，同时 process_media_player_call 函数对 setDataSource 返回值做判断是否设置成功以及失败后抛出异常。

mediaplayer 中的 setDataSource 函数分为三种，分别对应播放文件、网络和流三种情况，现在只看播放文件的情况：

```cpp
status_t MediaPlayer::setDataSource(int fd, int64_t offset, int64_t length)
{
    ALOGV("setDataSource(%d, %" PRId64 ", %" PRId64 ")", fd, offset, length);
    status_t err = UNKNOWN_ERROR;
    // 获取 MediaPlayerService 接口
    const sp<IMediaPlayerService>& service(getMediaPlayerService());
    if (service != 0) {
    	// 获取 MediaPlayer 接口
        sp<IMediaPlayer> player(service->create(this, mAudioSessionId));
        // 设置数据源
        if ((NO_ERROR != doSetRetransmitEndpoint(player)) ||
            (NO_ERROR != player->setDataSource(fd, offset, length))) {
           player.clear();
        }
        err = attachNewPlayer(player);
    }
    return err;
}
```

这个函数分为三个部分，逐个分析。

#### 3.2.1 获取 MediaPlayerService 接口

这里先是调用了 getMediaPlayerService 方法获取到一个 IMediaPlayerService

```cpp

/*static*/const sp<IMediaPlayerService> &IMediaDeathNotifier::getMediaPlayerService()
{
    ALOGV("getMediaPlayerService");
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService == 0) {
    	// 获取 servicemanager
        sp<IServiceManager> sm = defaultServiceManager();
        sp<IBinder> binder;
        do {
            // 获取对应的 binder
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
        // asInterface 获取 binder 中的 remote 对象
        sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    ALOGE_IF(sMediaPlayerService == 0, "no media player service!?");
    return sMediaPlayerService;
}
```

IServiceManager 是 IInterface 类型，和用 AIDL 生成的接口相同，IInterface 中包含了 binder 和 remote 两个东西。在 ndk 中对应的是 BnInterface 和 BpInterface。在 Java 中，本地 Client 通过 ServiceConnection 中的 onServiceConnected(ComponentName name, IBinder service) 方法获得 Service 中在 onBind 时候返回的 binder，同理，这里通过 ServiceManager 的 getService 获得和 media.player 这个 Service 通信用的 binder。

到这里为止我们只拿到了 media.player Service 的 binder，要想调用接口中的方法还需要通过 asInterface 方法来获得与之对应的 IInterface 接口。这里的 interface_cast<IMediaPlayerService>(binder) 方法就可以获得对应的接口，那么 [interface_cast](https://android.googlesource.com/platform/frameworks/native/+/jb-mr1-dev/include/binder/IInterface.h) 是怎么做到的呢，其实很简单，利用了模板封装了 asInterface 的操作：

```cpp
template<typename INTERFACE> inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```

现在我们就拿到了 media.player Service 的接口。

看到这里会有一个疑问，这个 media.player 的 Service 是什么时候启动的呢。我们根据 media.player 这个线索找到了下面的代码：

```cpp

void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService());
}

int main(int argc __unused, char** argv)
{
    ...
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    ALOGI("ServiceManager: %p", sm.get());
    AudioFlinger::instantiate();
    // 就是这里啦
    MediaPlayerService::instantiate();
    // 照相机
    CameraService::instantiate();
    // 音频
    AudioPolicyService::instantiate();
    // 语音识别
    SoundTriggerHwService::instantiate();
    registerExtensions();
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}

```

其中 MediaPlayerService 位于 [MediaPlayerService.cpp](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/libmediaplayerservice/MediaPlayerService.cpp) 中，而 main 函数位于 [main_mediaserver.cpp](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/mediaserver/main_mediaserver.cpp)。还记得在本文最开始的那张图么，其中的 mediaserver 对应的就是这个。它是在开机的时候启动的，被写在了启动脚本中：

![mediaserver 启动脚本](http://7xisp0.com1.z0.glb.clouddn.com/media_server_service.png)

这样在系统启动的时候这个进程就开启了，同时里面的 Service 也就启动了。

#### 3.2.2 获取 MediaPlayer 接口

service->create(this, mAudioSessionId) 方法通过 IPC 的方式调用 [MediaPlayerService](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/libmediaplayerservice/MediaPlayerService.cpp) 中的 create 方法获得 [IMediaPlayer](https://android.googlesource.com/platform/frameworks/av/+/master/media/libmedia/IMediaPlayer.cpp)，IMediaPlayer 从名字就可以看出是一个 IInterface，所以 MediaPlayer 也是通过 IPC 来调用的。先看这个方法做了什么：

```cpp
sp<IMediaPlayer> MediaPlayerService::create(const sp<IMediaPlayerClient>& client, int audioSessionId)
{
    pid_t pid = IPCThreadState::self()->getCallingPid();
    int32_t connId = android_atomic_inc(&mNextConnId);
    // MediaPlayerClient
    sp<Client> c = new Client(
            this, pid, connId, client, audioSessionId,
            IPCThreadState::self()->getCallingUid());
    ALOGV("Create new client(%d) from pid %d, uid %d, ", connId, pid,
         IPCThreadState::self()->getCallingUid());
    wp<Client> w = c;
    {
        Mutex::Autolock lock(mLock);
        // 管理 Client
        mClients.add(w);
    }
    return c;
}
```

首先调用的时候传的参数是 MediaPlayer 本身，MediaPlayer 除了继承 IMediaDeathNotifier 同时还继承了 BnMediaPlayerClient，而 BnMediaPlayerClient 又继承了 BnInterface<IMediaPlayerClient>，所以这里的参数列表是一个 client 引用。其中 Client 的实现也在 MediaPlayerService.cpp 这个文件中，他的构造函数用来保存这些对象：

```cpp
MediaPlayerService::Client::Client(
        const sp<MediaPlayerService>& service, pid_t pid,
        int32_t connId, const sp<IMediaPlayerClient>& client,
        int audioSessionId, uid_t uid)
{
    ALOGV("Client(%d) constructor", connId);
    mPid = pid;
    mConnId = connId;
    mService = service;
    mClient = client;
    mLoop = false;
    mStatus = NO_INIT;
    mAudioSessionId = audioSessionId;
    mUID = uid;
    mRetransmitEndpointValid = false;
    mAudioAttributes = NULL;
#if CALLBACK_ANTAGONIZER
    ALOGD("create Antagonizer");
    mAntagonizer = new Antagonizer(notify, this);
#endif
}
```

最后 create 方法返回一个 MediaPlayer 的接口，这个接口通过 IPC 用来调用 Client 中的函数。

#### 3.2.3 设置数据源

经过了之前的折腾，我们先拿到了 MediaPlayerService 的接口，通过 MediaPlayerService 的接口又拿到了 MediaPlayer 的接口，接下来就要进行这个函数的最终目的**设置数据源**。

同样是通过 IPC 调用了 MediaPlayer 的 setDataSource，定义如下：

```cpp
status_t MediaPlayerService::Client::setDataSource(int fd, int64_t offset, int64_t length)
{
    // 前面是一些输出要设置的数据源的信息的 log
	...
    // 获取播放器类型
    player_type playerType = MediaPlayerFactory::getPlayerType(this, fd, offset, length);
    // 创建播放器
    sp<MediaPlayerBase> p = setDataSource_pre(playerType);
    if (p == NULL) {
        return NO_INIT;
    }
    // now set data source
    setDataSource_post(p, p->setDataSource(fd, offset, length));
    return mStatus;
}

```

首先是获取播放器类型，播放器类型定义在 [MediaPlayerInterface.h](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/include/media/MediaPlayerInterface.h) 中：

```cpp
enum player_type {
    PV_PLAYER = 1,
    SONIVOX_PLAYER = 2,
    STAGEFRIGHT_PLAYER = 3,
    NU_PLAYER = 4,
    // Test players are available only in the 'test' and 'eng' builds.
    // The shared library with the test player is passed passed as an
    // argument to the 'test:' url in the setDataSource call.
    TEST_PLAYER = 5,
};

```

下面说下这几种类型是做什么的，其中的每一个都对应一个工厂来创建对应的 Player，由于 PV_PLAYER 已经被抛弃了，所以在 5.1 的源码里并没有出现它

1.**PV_PLAYER**    这个类型是 Android 最初采用的 OpenCore，由于太臃肿已经被抛弃

2.**SONIVOX_PLAYER**   用来处理 midi 相关

3.**NU_PLAYER**    全能型，在 5.x 上处于可选

4.**STAGEFRIGHT_PLAYER**    5.x 之前的主力即 awesome player ，可以胜任除 midi 外全部的工作

5.**TEST_PLAYER**    测试用

这些 player 都是由对应的 factory 创建的，对应的实现在 [MediaPlayerFactory.cpp](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/libmediaplayerservice/MediaPlayerFactory.cpp) 中，其中的代码比较简单，这里就不分析了，主要是匹配不同类型对应不同的分数，然后选取分高的 player 创建。当前的主力是 STAGEFRIGHT_PLAYER 也就是 awesome player，而 NU_PLAYER  是未来的主力，从 Android M 目前的[源码](https://android.googlesource.com/platform/frameworks/av/+/android-m-preview-2/media/libmediaplayerservice/MediaPlayerFactory.cpp)中也可以看出代码中只剩下了 NU_PLAYER 和 STAGEFRIGHT_PLAYER，其中 NU_PLAYER 负责网络和流的播放，STAGEFRIGHT_PLAYER 负责有 DRM 和文件的播放。

这里我们就以目前的主力 STAGEFRIGHT_PLAYER 播放器继续往下分析。获取到播放器类型后就到了**创建播放器**

```cpp

sp<MediaPlayerBase> MediaPlayerService::Client::setDataSource_pre(player_type playerType)
{
    ...
    // create the right type of player
    sp<MediaPlayerBase> p = createPlayer(playerType);
    if (p == NULL) {
        return p;
    }
	...
    return p;
}
sp<MediaPlayerBase> MediaPlayerService::Client::createPlayer(player_type playerType)
{
    // mPlayer 是当前已经有的 player，如果是刚创建这个对象，
    // 那么 player 是 null 需要创建新的 player
    // 如果创建过了了则对比下需要用到的 player 类型，避免重复创建
    sp<MediaPlayerBase> p = mPlayer;
    if ((p != NULL) && (p->playerType() != playerType)) {
        ALOGV("delete player");
        p.clear();
    }
    if (p == NULL) {
        p = MediaPlayerFactory::createPlayer(playerType, this, notify);
    }
	...
    return p;
}

```

这里从 MediaPlayerFactory 创建了对应的播放器，之后使用创建好的 player 设置数据源：

```cpp
status_t StagefrightPlayer::setDataSource(int fd, int64_t offset, int64_t length) {
    ALOGV("setDataSource(%d, %lld, %lld)", fd, offset, length);
    return mPlayer->setDataSource(dup(fd), offset, length);
}
```

这里的 mPlayer 就是 AwesomePlayer，AwesomePlayer 在 StagefrightPlayer 类的构造函数中创建：

```cpp
StagefrightPlayer::StagefrightPlayer(): mPlayer(new AwesomePlayer) {
    ALOGV("StagefrightPlayer");
    mPlayer->setListener(this);
}
```

从 [StagefrightPlayer](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/libmediaplayerservice/StagefrightPlayer.cpp) 的源码可以看到对象中的 AwesomePlayer 来完成。代码如下：

```cpp
status_t AwesomePlayer::setDataSource(
        int fd, int64_t offset, int64_t length) {
    Mutex::Autolock autoLock(mLock);
	// 重置播放器状态
    reset_l();
	// 封装成 DataSource
    sp<DataSource> dataSource = new FileSource(fd, offset, length);
	...
    return setDataSource_l(dataSource);
}

status_t AwesomePlayer::setDataSource_l(const sp<DataSource> &dataSource) {
    // 通过 datasource 创建分离器
    sp<MediaExtractor> extractor = MediaExtractor::Create(dataSource);
    if (extractor == NULL) {
        return UNKNOWN_ERROR;
    }
    if (extractor->getDrmFlag()) {
        checkDrmStatus(dataSource);
    }
	// 为分离器设置数据源
    return setDataSource_l(extractor);
}
```

这个分离器其实和 ffpmeg 中的 demuxer 一样，作用是将数据中的音频部分和视频部分分开，然后获取到对应的类型，调用对应的解码器进行解码：

```cpp
status_t AwesomePlayer::setDataSource_l(const sp<MediaExtractor> &extractor) {
    // Attempt to approximate overall stream bitrate by summing all
    // tracks' individual bitrates, if not all of them advertise bitrate,
    // we have to fail.
    int64_t totalBitRate = 0;
    mExtractor = extractor;
    for (size_t i = 0; i < extractor->countTracks(); ++i) {
        sp<MetaData> meta = extractor->getTrackMetaData(i);
        int32_t bitrate;
        if (!meta->findInt32(kKeyBitRate, &bitrate)) {
            const char *mime;
            CHECK(meta->findCString(kKeyMIMEType, &mime));
            ALOGV("track of type '%s' does not publish bitrate", mime);
            totalBitRate = -1;
            break;
        }
        // 总的码率
        totalBitRate += bitrate;
    }
    // metadata 就是数据中的元信息
    sp<MetaData> fileMeta = mExtractor->getMetaData();
    if (fileMeta != NULL) {
        int64_t duration;
        if (fileMeta->findInt64(kKeyDuration, &duration)) {
            // 长度
            mDurationUs = duration;
        }
    }
    mBitrate = totalBitRate;
    bool haveAudio = false;
    bool haveVideo = false;
    for (size_t i = 0; i < extractor->countTracks(); ++i) {
        sp<MetaData> meta = extractor->getTrackMetaData(i);
        const char *_mime;
        CHECK(meta->findCString(kKeyMIMEType, &_mime));
        String8 mime = String8(_mime);
        // 视频
        if (!haveVideo && !strncasecmp(mime.string(), "video/", 6)) {
            setVideoSource(extractor->getTrack(i));
            haveVideo = true;
			...
        // 音频
        } else if (!haveAudio && !strncasecmp(mime.string(), "audio/", 6)) {
            setAudioSource(extractor->getTrack(i));
            haveAudio = true;
			...
            // ogg
            if (!strcasecmp(mime.string(), MEDIA_MIMETYPE_AUDIO_VORBIS)) {
                // Only do this for vorbis audio, none of the other audio
                // formats even support this ringtone specific hack and
                // retrieving the metadata on some extractors may turn out
                // to be very expensive.
                sp<MetaData> fileMeta = extractor->getMetaData();
                int32_t loop;
                if (fileMeta != NULL
                        && fileMeta->findInt32(kKeyAutoLoop, &loop) && loop != 0) {
                    modifyFlags(AUTO_LOOPING, SET);
                }
            }
        } else if (!strcasecmp(mime.string(), MEDIA_MIMETYPE_TEXT_3GPP)) {
            addTextSource_l(i, extractor->getTrack(i));
        }
    }
	...
    return OK;
}
```

这里通过分离器来确定数据源中是音频还是视频，然后对 player 的 videotrack 和 audiotrack 进行相对应的设置。

到此为止 setDataSource 的流程就算是走完了，如果继续往下分析的话就到了视频的编解码的知识了。下面先用几张图来梳理一下这个过程（画的不是很规范）。

**流程图：**

![setDataSource流程图](http://7xisp0.com1.z0.glb.clouddn.com/mediaplayer_set_data_flow.png)

**类图：**

![mediaplayer 类图](http://7xisp0.com1.z0.glb.clouddn.com/mediaplayer_class_diagram.png)

类图中的 Client 是 MediaPlayerService 的内部类。MediaPlayerService 通过 IPC 和 MediaPlayerService 通信获得 Client，Client 中包含了 StagefrightPlayer，而 StagefrightPlayer 最终通过调用 AwesomePlayer 对应的方法。

### 3.3 Prepare

通过之前的经验我们可以很快的知道 prepare 应该对应的是 AwesomePlayer 中的 prepare：

```cpp
status_t AwesomePlayer::prepare() {
    ...
    return prepare_l();

}

status_t AwesomePlayer::prepare_l() {
    ...
    status_t err = prepareAsync_l();
    ...
    return mPrepareResult;
}

status_t AwesomePlayer::prepareAsync_l() {
    if (mFlags & PREPARING) {
        return UNKNOWN_ERROR;  // async prepare already pending
    }
    if (!mQueueStarted) {
        mQueue.start();
        mQueueStarted = true;
    }
    modifyFlags(PREPARING, SET);
    mAsyncPrepareEvent = new AwesomeEvent(
            this, &AwesomePlayer::onPrepareAsyncEvent);
    mQueue.postEvent(mAsyncPrepareEvent);
    return OK;
}
```

mQueue 是 [TimedEventQueue](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/libstagefright/TimedEventQueue.cpp)，TimedEventQueue 和 Handler 很相似，使用 pthread 和 队列来管理消息。在这里通过异步方式回调 onPrepareAsyncEvent 方法：

```cpp
void AwesomePlayer::onPrepareAsyncEvent() {
    Mutex::Autolock autoLock(mLock);
    beginPrepareAsync_l();
}
void AwesomePlayer::beginPrepareAsync_l() {
	// 取消 prepare
    if (mFlags & PREPARE_CANCELLED) {
        ALOGI("prepare was cancelled before doing anything");
        abortPrepare(UNKNOWN_ERROR);
        return;
    }
	// 网络媒体
    if (mUri.size() > 0) {
        status_t err = finishSetDataSource_l();
        if (err != OK) {
            abortPrepare(err);
            return;
        }
    }
	// 是否包含视频
    if (mVideoTrack != NULL && mVideoSource == NULL) {
    	// 初始化视频解码器
        status_t err = initVideoDecoder();
        if (err != OK) {
            abortPrepare(err);
            return;
        }
    }
	// 是否包含音频
    if (mAudioTrack != NULL && mAudioSource == NULL) {
    	// 初始化音频解码器
        status_t err = initAudioDecoder();
        if (err != OK) {
            abortPrepare(err);
            return;
        }
    }
    modifyFlags(PREPARING_CONNECTED, SET);
	// 不同的播放源类型
    if (isStreamingHTTP()) {
    	// 网络流媒体
        postBufferingEvent_l();
    } else {
    	// 本地媒体
        finishAsyncPrepare_l();
    }
}

void AwesomePlayer::finishAsyncPrepare_l() {
    if (mIsAsyncPrepare) {
        if (mVideoSource == NULL) {
            notifyListener_l(MEDIA_SET_VIDEO_SIZE, 0, 0);
        } else {
            notifyVideoSize_l();
        }
        notifyListener_l(MEDIA_PREPARED);
    }
    // 设置 player 的状态
    mPrepareResult = OK;
    modifyFlags((PREPARING|PREPARE_CANCELLED|PREPARING_CONNECTED), CLEAR);
    modifyFlags(PREPARED, SET);
    mAsyncPrepareEvent = NULL;
    // 同步线程之间的状态
    mPreparedCondition.broadcast();
	// mAudioTearDown 默认为 false，当暂停的时候通过回调方法将其改为 true
    if (mAudioTearDown) {
        if (mPrepareResult == OK) {
            if (mExtractorFlags & MediaExtractor::CAN_SEEK) {
                seekTo_l(mAudioTearDownPosition);
            }
            if (mAudioTearDownWasPlaying) {
                modifyFlags(CACHE_UNDERRUN, CLEAR);
                play_l();
            }
        }
        mAudioTearDown = false;
    }
}
```

经过一番设置后，prepare 的过程也算是完成了。这一部分主要是对 player 的状态进行设置，通过消息机制让让调用端也知道这边的 player 状态。

### 3.4 start

Java 代码这边的 start 方法对应的是 [StagefrightPlayer](https://android.googlesource.com/platform/frameworks/av/+/android-5.1.1_r18/media/libmediaplayerservice/StagefrightPlayer.cpp) 中的 start，其中又调用了 player 的 play 方法

```cpp
status_t StagefrightPlayer::start() {
    ALOGV("start");
    return mPlayer->play();
}
```

对应的是 AwesomePlayer 中的 play 方法：

```cpp
status_t AwesomePlayer::play() {
    ATRACE_CALL();
    Mutex::Autolock autoLock(mLock);
    modifyFlags(CACHE_UNDERRUN, CLEAR);
    return play_l();
}
status_t AwesomePlayer::play_l() {
    modifyFlags(SEEK_PREVIEW, CLEAR);
	// 如果正在播放，则什么也不做
    if (mFlags & PLAYING) {
        return OK;
    }
    mMediaRenderingStartGeneration = ++mStartGeneration;
    // 如果没有之前没有调用 prepare，这里会帮你调用一次，顺便我还做了个实验，create 之后 直接 start 是可以播放的，并没有报错什么的。
    if (!(mFlags & PREPARED)) {
        status_t err = prepare_l();
        if (err != OK) {
            return err;
        }
    }
    modifyFlags(PLAYING, SET);
    modifyFlags(FIRST_FRAME, SET);
    if (mDecryptHandle != NULL) {
        int64_t position;
        getPosition(&position);
        mDrmManagerClient->setPlaybackStatus(mDecryptHandle,
                Playback::START, position / 1000);
    }
    if (mAudioSource != NULL) {
        if (mAudioPlayer == NULL) {
            createAudioPlayer_l();
        }
        CHECK(!(mFlags & AUDIO_RUNNING));
        if (mVideoSource == NULL) {
            // We don't want to post an error notification at this point,
            // the error returned from MediaPlayer::start() will suffice.
            status_t err = startAudioPlayer_l(false /* sendErrorNotification */);
            // 是否可以硬解音频
            if ((err != OK) && mOffloadAudio) {
                ALOGI("play_l() cannot create offload output, fallback to sw decode");
                int64_t curTimeUs;
                getPosition(&curTimeUs);
                delete mAudioPlayer;
                mAudioPlayer = NULL;
                // if the player was started it will take care of stopping the source when destroyed
                if (!(mFlags & AUDIOPLAYER_STARTED)) {
                    mAudioSource->stop();
                }
                modifyFlags((AUDIO_RUNNING | AUDIOPLAYER_STARTED), CLEAR);
                mOffloadAudio = false;
                mAudioSource = mOmxSource;
                if (mAudioSource != NULL) {
                    err = mAudioSource->start();
                    if (err != OK) {
                        mAudioSource.clear();
                    } else {
                        mSeekNotificationSent = true;
                        if (mExtractorFlags & MediaExtractor::CAN_SEEK) {
                            seekTo_l(curTimeUs);
                        }
                        createAudioPlayer_l();
                        // 播放音频
                        err = startAudioPlayer_l(false);
                    }
                }
            }
            if (err != OK) {
                delete mAudioPlayer;
                mAudioPlayer = NULL;
                modifyFlags((PLAYING | FIRST_FRAME), CLEAR);
                if (mDecryptHandle != NULL) {
                    mDrmManagerClient->setPlaybackStatus(mDecryptHandle, Playback::STOP, 0);
                }
                return err;
            }
        }
    }
    // timesource 用于视音频同步，通常来说是根据 audio 中的 timesource 为基准，
    // 这里判断下是否有音频时间和音频播放器，如果没有，则采用系统的时间
    if (mTimeSource == NULL && mAudioPlayer == NULL) {
        mTimeSource = &mSystemTimeSource;
    }
    // 播放视频
    if (mVideoSource != NULL) {
        // Kick off video playback
        postVideoEvent_l();
        if (mAudioSource != NULL && mVideoSource != NULL) {
            postVideoLagEvent_l();
        }
    }
    // 流的结尾
    if (mFlags & AT_EOS) {
        // Legacy behaviour, if a stream finishes playing and then
        // is started again, we play from the start...
        seekTo_l(0);
    }
    uint32_t params = IMediaPlayerService::kBatteryDataCodecStarted
        | IMediaPlayerService::kBatteryDataTrackDecoder;
    if ((mAudioSource != NULL) && (mAudioSource != mAudioTrack)) {
        params |= IMediaPlayerService::kBatteryDataTrackAudio;
    }
    if (mVideoSource != NULL) {
        params |= IMediaPlayerService::kBatteryDataTrackVideo;
    }
    addBatteryData(params);
    if (isStreamingHTTP()) {
        postBufferingEvent_l();
    }
    return OK;
}
```
视频的播放是通过不断的 postVideoEvent_l 来实现画面的播放，postVideoEvent_l 会向队列中 post 一个 videoEvent，这个 event 最终又会触发 AwesomePlayer::onVideoEvent 的方法， onVideoEvent 方法中又会调用 postVideoEvent_l，形成循环，最终实现视频的播放。startAudioPlayer_l 最终会调用到 [AudioPlayer](https://android.googlesource.com/platform/frameworks/av/+/master/media/libstagefright/AudioPlayer.cpp) 播放音频。

播放的过程到这里就结束了，接下来暂停和结束的部分就相对简单了。

### 3.5 pause
pause 对应到 AwesomePlayer 中是 pause_l 这个方法：
```cpp
status_t AwesomePlayer::pause_l(bool at_eos) {
	...
	// 通知客户端播放器暂停
    notifyListener_l(MEDIA_PAUSED);
    mMediaRenderingStartGeneration = ++mStartGeneration;
	// cancel 播放视频的队列
    cancelPlayerEvents(true /* keepNotifications */);
    if (mAudioPlayer != NULL && (mFlags & AUDIO_RUNNING)) {
        // If we played the audio stream to completion we
        // want to make sure that all samples remaining in the audio
        // track's queue are played out.
        mAudioPlayer->pause(at_eos /* playPendingSamples */);
        // send us a reminder to tear down the AudioPlayer if paused for too long.
        if (mOffloadAudio) {
            postAudioTearDownEvent(kOffloadPauseMaxUs);
        }
        modifyFlags(AUDIO_RUNNING, CLEAR);
    }
    if (mFlags & TEXTPLAYER_INITIALIZED) {
        mTextDriver->pause();
        modifyFlags(TEXT_RUNNING, CLEAR);
    }
    modifyFlags(PLAYING, CLEAR);
    if (mDecryptHandle != NULL) {
        mDrmManagerClient->setPlaybackStatus(mDecryptHandle,
                Playback::PAUSE, 0);
    }
	...
    return OK;
}
```

### 3.6 stop
然而 stop 和 pause 并没什么区别，代码中的注释也是挺有意思
```cpp
status_t StagefrightPlayer::stop() {
    ALOGV("stop");
    return pause();  // what's the difference?
}
status_t StagefrightPlayer::pause() {
    ALOGV("pause");

    return mPlayer->pause();
}
```

### 3.7 release
MeidaPlayer 对应 jni 中的 android_media_MediaPlayer_release 方法：
```cpp
static void
android_media_MediaPlayer_release(JNIEnv *env, jobject thiz)
{
    ALOGV("release");
    decVideoSurfaceRef(env, thiz);
    sp<MediaPlayer> mp = setMediaPlayer(env, thiz, 0);
    if (mp != NULL) {
        // this prevents native callbacks after the object is released
        mp->setListener(0);
        mp->disconnect();
    }
}
```
setListener 注释中说的很明白了，就是把 listener 置空。diconnect 这里是断开了了和 mediaserver 的连接。
```cpp
void MediaPlayer::disconnect()
{
    ALOGV("disconnect");
    sp<IMediaPlayer> p;
    {
        Mutex::Autolock _l(mLock);
        p = mPlayer;
        mPlayer.clear();
    }
    if (p != 0) {
        p->disconnect();
    }
}
```

## 4.总结
MediaPlayer 整体上的流程就是这些，其中相对复杂的地方集中在编解码和不同 player 的使用上。编解码相关的主要是 OMX 的硬解和软解，player 主要是不同的 Android 版本对不同的 player 优先使用不同。


## 5.参考
[Media Playback](http://developer.android.com/guide/topics/media/mediaplayer.html)

[MediaPlayer](http://developer.android.com/reference/android/media/MediaPlayer.html)

[Media](https://source.android.com/devices/media/index.html)

[深入理解 Android I](http://wiki.jikexueyuan.com/project/deep-android-v1/binder.html)