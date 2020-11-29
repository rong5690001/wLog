MediaPlayerService是多媒体框架中一个非常重要的服务，从前面的内容中我们可以理解MediaPlayer是客户端，MediaPlayerService和MediaPlayerService::Client是服务器端。MediaPlayerService实现IMediaPlayerService定义的业务逻辑，其主要功能是根据MediaPlayer::setDataSource输入的URL调用create函数创建对应的player。MediaPlayerService::Client实现IMediaPlayer定义的业务逻辑，其主要功能包括start、stop、pause、resume……其是通过调用MediaPlayerService create的player中的对应方法来实现具体功能的。

## 一、Client/Server通过IPC的通信流程图

![screenshot_20201129_204622](https://tva1.sinaimg.cn/large/0081Kckwgy1gl6c5hff1ij31400mi0yk.jpg)

<center>图3-1 MediaPlayer和MediaPlayerService通过Binder通信</center>

从图3-1中可以总结出如下几点。

- MediaPlayer是客户端，也就是我们所说的C/S模型中的C端，即Client。
- MediaPlayerService和MediaPlayerService::Client是服务器端，也就是我们所说的C/S模型中的S端，即Server端。
-  MediaPlayerService实现IMediaPlayerService定义的业务逻辑，其主要功能是根据MediaPlayer::setDataSource输入的URL调用create函数创建对应的player。
- MediaPlayerService::Client实现IMediaPlayer定义的业务逻辑，其主要功能包括start、stop、pause、resume……其是通过调用MediaPlayerService create的player中的对应方法来实现具体功能的。
-  通过Transact函数可以向远端的IBinder对象发出调用，通过onTransact函数可以使你自己的远程对象能够响应接收到的调用。

## 二、相关联的类图

![screenshot_20201129_205639](https://tva1.sinaimg.cn/large/0081Kckwgy1gl6c8yxuv2j30mi1400xe.jpg)

<center>图3-2 IPC通信相关联的类关系（1）</center>

从图3-2可以总结为如下几点:

- BpXXX和BnXXX都派生自IXXX，那IXXX又是做什么的呢？这里可以理解为定义业务逻辑，我们此前分析IMediaPlayerClient的作用时，也介绍过。但BpXXX与BnXXX中的实现方式不同。
- 在BpXXX中，把对应的binder_transaction_data打包之后，通过BpRefBase中的mRemote(BpBinder)发送出去，并等待结果。
- 在BnXXX中，实现对应的业务逻辑，通过调用BnXXX派生类中的方法来实现，如MediaPlayerService::Client。

从图3-3可以看出，IBinder用于进行进程间通信。

- 图3-2中的BpRefBase中有一个remote函数用来与Binder驱动交互使用。
- Binder是用来从Binder驱动中接收相关请求并进行相关处理的。
- BpBinder和BinderDriver进行互通。

![screenshot_20201129_210133](https://tva1.sinaimg.cn/large/0081Kckwgy1gl6ce7n09uj30mi140n1t.jpg)

<center>图3-3 IPC通信相关联的类关系（2）</center>

## 三、产生过程

在了解MediaPlayerService之前，先了解一下IMediaPlayerService.cpp，在C++ 6.0源码中其处于frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp中：

```c++
#ifndef ANDROID_IMEDIAPLAYERSERVICE_H
#define ANDROID_IMEDIAPLAYERSERVICE_H

namespace android {

struct IHDCP;
class IMediaCodecList;
struct IMediaHTTPService;
class IMediaRecorder;
class IOMX;
class IRemoteDisplay;
class IRemoteDisplayClient;
struct IStreamSource;

class IMediaPlayerService: public IInterface
{
public:
    DECLARE_META_INTERFACE(MediaPlayerService);

    virtual sp<IMediaRecorder> createMediaRecorder(const String16 &opPackageName) = 0;
    virtual sp<IMediaMetadataRetriever> createMetadataRetriever() = 0;
    virtual sp<IMediaPlayer> create(const sp<IMediaPlayerClient>& client,
            audio_session_t audioSessionId = AUDIO_SESSION_ALLOCATE) = 0;
    virtual sp<IOMX>            getOMX() = 0;
    virtual sp<IHDCP>           makeHDCP(bool createEncryptionModule) = 0;
    virtual sp<IMediaCodecList> getCodecList() const = 0;
		//省略部分代码
// ----------------------------------------------------------------------------

class BnMediaPlayerService: public BnInterface<IMediaPlayerService>
{
public:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
};

}; // namespace android

#endif // ANDROID_IMEDIAPLAYERSERVICE_H
```

可以看出这里定义了一些常规播放控制接口，接下来开始了解MediaPlayerService，首先找到入口，在frameworks/base/media/mediaserver/main_mediaserver.cpp中：

```c++
int main(int argc __unused, char** argv)
{
    //省略部分代码
    if (doLog && (childPid = fork()) != 0) {
        // media.log service
        //prctl(PR_SET_NAME, (unsigned long) "media.log", 0, 0, 0);
        // unfortunately ps ignores PR_SET_NAME for the main thread, so use this ugly hack
        strcpy(argv[0], "media.log");
        //获利ProcessState，在构造函数中打开Binder驱动
        sp<ProcessState> proc(ProcessState::self());
        MediaLogService::instantiate();
        ProcessState::self()->startThreadPool();
        for (;;) {
            //省略部分代码
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder = sm->getService(String16("media.log"));
            if (binder != 0) {
                Vector<String16> args;
                binder->dump(-1, args);
            }
            //省略部分代码
        }
    } else {
        //其他的Service实例化
        InitializeIcuOrDie();
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        MediaPlayerService::instantiate();
        ResourceManagerService::instantiate();
        CameraService::instantiate();
        AudioPolicyService::instantiate();
        SoundTriggerHwService::instantiate();
        RadioService::instantiate();
        registerExtensions();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }
}
```

接着看看defaultServiceManager函数，代码如下：

