## 一、状态图及生命周期

MediaPlayer类用于视频/音频文件的播放控制。本节主要覆盖MediaPlayer如下知识点。

- MediaPlayer的状态图

- Idle状态

- End状态

- Error状态

- Initialized状态

- Prepared状态

- Preparing状态

- Started状态

- Paused状态

- Stopped状态

- PlaybackCompleted状态

### 1.1、MediaPlayer的状态图

MediaPlayer用于控制视频/音频文件及流的播放，由状态机进行管理。图2-1显示了MediaPlayer状态周期。

![screenshot_20201129_204622](https://tva1.sinaimg.cn/large/0081Kckwgy1gl6bygoctij31400mi0yk.jpg)

<center>图2-1 MediaPlayer状态周期</center>

### 1.2、Idle状态及End状态

在MediaPlayer创建实例或者调用reset函数后，播放器就被创建了，这时处于Idle（就绪）状态，调用release函数后，就会变成End（结束）状态，在这两种状态之间的就是MediaPlayer的生命周期。

### 1.3、Error状态

在构造一个新MediaPlayer或者调用reset函数后，上层应用程序调用getCurrentPosition、getVideoHeight、getDuration、getVideoWidth、setAudioStreamType(int)、setLooping(boolean)、setVolume(float,float)、pause、start、stop、seekTo(int)、prepare、prepareAsync这些函数会出错。如果调用reset函数后再调用它们，用户提供的回调函数OnErrorListener.onError将触发MediaPlayer状态到Error（错误）状态，所以一旦不再使用MediaPlayer，就需要调用release函数，以便MediaPlayer资源得到合理释放。

当MediaPlayer处于End（结束）状态时，它将不能再被使用，这时不能再回到MediaPlayer的其他状态，因为本次生命周期已经终止。

一旦有错误，MediaPlayer会进入Error（错误）状态，为了重新使用MediaPlayer，调用reset函数，这时将重新恢复到Idle（就绪）状态，所以需要给MediaPlayer设置错误监听，出错后就可以从播放器内部返回的信息中找到错误原因。

### 1.4、Initialized状态

当调用setDataSource(FileDescriptor)、setDataSource(String)、setDataSource(Context, Uri)、setDataSource(FileDescriptor, long, long)其中一个函数时，将传递MediaPlayer的Idle状态变成Initialized（初始化）状态，如果setDataSource在非Idle状态时调用，会抛出IllegalStateException异常。当重载setDataSource时，需要抛出IllegalArgumentException和IOException这两个异常。

### 1.5、Prepared状态

MediaPlayer有两种途径到达Prepared状态，一种是同步方式，另一种是异步方式。同步方式主要使用本地音视频文件，异步方式主要使用网络数据，需要缓冲数据。调用prepare（同步函数）将传递MediaPlayer的Initialized状态变成Prepared状态，或者调用prepareAsync（异步函数）将传递MediaPlayer的Initialized状态变成Preparing状态，最后到Prepared状态。如果应用层事先注册过setOnPreparedListener，播放器内部将回调用户设置的OnPreparedListener中的onPrepared回调函数。注意，Preparing是一个瞬间状态（可理解为时间比较短）。

### 1.6、Started状态

在MediaPlayer进入Prepared状态后，上层应用即可设置一些属性，如音视频的音量、screenOnWhilePlaying、looping等。在播放控制开始之前，必须调用start函数并成功返回，MediaPlayer的状态开始由Prepared状态变成Started状态。当处于Started状态时，如果用户事先注册过setOnBufferingUpdateListener，播放器内部会开始回调OnBufferingUpdateListener.on BufferingUpdate，这个回调函数主要使应用程序保持跟踪音视频流的buffering（缓冲）status，如果MediaPlayer已经处于Started状态，再调用start函数是没有任何作用的。

### 1.7、Paused状态

MediaPlayer在播放控制时可以是Paused（暂停）和Stopped（停止）状态的，且当前的播放时进度可以被调整，当调用MediaPlayer.pause函数时，MediaPlayer开始由Started状态变成Paused状态，这个从Started状态到Paused状态的过程是瞬间的，反之在播放器内部是异步过程的。在状态更新并调用isPlaying函数前，将有一些耗时。已经缓冲过的数据流，也要耗费数秒。
当start函数从Paused状态恢复回来时，playback恢复之前暂停时的位置，接着开始播放，这时MediaPlayer的Paused状态又变成Started状态。如果MediaPlayer已经处于Paused状态，这时再调用pause函数是没有任何作用的，将保持Paused状态。

### 1.8、Stopped状态

当调用stop函数时，MediaPlayer无论正处于Started、Paused、Prepared或PlaybackCompleted中的哪种状态，都将进入Stopped状态。一旦处于Stopped状态，playback将不能开始，直到重新调用prepare或prepareAsync函数，且处于Prepared状态时才可以开始。
如果MediaPlayer已经处于Stopped状态了，这时再调用stop函数是没有任何作用的，将保持Stopped状态。
在Seek操作完成后，如果事先在MediaPlayer注册了setOnSeekCompleteListener，播放器内部将回调OnSeekComplete.onSeekComplete函数。当然seekTo函数也可以在其他状态下被调用，如Prepared、Paused及PlaybackCompleted状态。

### 1.9、PlaybackCompleted状态

当前播放的位置可以通过getCurrentPosition函数获取，通过getCurrentPosition函数，可以跟踪播放器的播放进度。
当MediaPlayer播放到数据流的末尾时，一次播放过程完成。在MediaPlayer中事先调用setLooping(boolean)并设置为true，表示循环播放，MediaPlayer依然处于Started状态。如果调用setLooping(boolean)并设置为false（表示不循环播放），并且事先在MediaPlayer上注册过setOnCompletionListener，播放器内部将回调OnCompletion.onCompletion函数，这就表明MediaPlayer开始进入PlaybackCompleted（播放完成）状态。当处于PlaybackCompleted状态时，调用start函数，将重启播放器从头开始播放数据。

## 二、从创建到setDataSource过程

### 2.1、从创建到setDisplay过程

MediaPlayer时序图一，如图2-2所示。

![screenshot_20201129_204358](https://tva1.sinaimg.cn/large/0081Kckwgy1gl6bwsvtzsj31400miqdf.jpg)

<center>图2-2 从create到setDisplay过程的时序图</center>

从时序图可以看到，通过getService从ServiceManager获取对应的MediaPlayerService，然后调用native_setup函数创建播放器，接着调用setDataSource把URL地址传入底层。当准备好后，通过setDisplay传入SurfaceHolder，以便将解码出的数据放到SurfaceHolder中的Surface。最后显示在SurfaceView上。

### 2.2、创建过程

当外部调用MediaPlayer.create(this, ＂http://www.xxx.mp4＂)时，进入MediaPlayer的创建过程：

![image-20201129202125306](https://tva1.sinaimg.cn/large/0081Kckwgy1gl6b8bzze9j30f40aoq5d.jpg)

```java
public static MediaPlayer create(Context context, Uri uri, SurfaceHolder holder,
            AudioAttributes audioAttributes, int audioSessionId) {

        try {
            MediaPlayer mp = new MediaPlayer();
          	//声音相关处理，若为空，就创建一个
            final AudioAttributes aa = audioAttributes != null ? audioAttributes :
                new AudioAttributes.Builder().build();
          	//设置声音属性
            mp.setAudioAttributes(aa);
          	//设置声音的会话ID，视频和音频是分开渲染的
            mp.setAudioSessionId(audioSessionId);
          	//从这里开始setDataSource。uri即统一资源标识符
            mp.setDataSource(context, uri);
          	//判断SurfaceHolder是否为空，这是一个控制器，Surface的控制器，用来操纵							//Surface，处理它在Canvas上作画的效果和动画，控制表面、大小、像素等
            if (holder != null) {
              	//给Surface设置一个控制器
                mp.setDisplay(holder);
            }
          	//准备开始
            mp.prepare();
            return mp;
        } catch (IOException ex) {
            Log.d(TAG, "create failed:", ex);
            // fall through
        } catch (IllegalArgumentException ex) {
            Log.d(TAG, "create failed:", ex);
            // fall through
        } catch (SecurityException ex) {
            Log.d(TAG, "create failed:", ex);
            // fall through
        }

        return null;
    }
```

再来看看new MediaPlayer方法内部：

```java
public MediaPlayer() {
        super(new AudioAttributes.Builder().build(),
                AudioPlaybackConfiguration.PLAYER_TYPE_JAM_MEDIAPLAYER);

        Looper looper;
  			//获取Looper，优先获取当前线程的Looper，为空的话获取主线程的Looper
        if ((looper = Looper.myLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else if ((looper = Looper.getMainLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else {
            mEventHandler = null;
        }
				//时间数据容器，一般provider都是和数据联系起来的，如videoProvider、									//ContentProvider
        mTimeProvider = new TimeProvider(this);
        mOpenSubtitleSources = new Vector<InputStream>();

        /* Native setup requires a weak reference to our object.
         * It's easier to create it here than in C++.
         */
        native_setup(new WeakReference<MediaPlayer>(this));

        baseRegisterPlayer();
    }
```

接下来看Native层如何创建一个MediaPlayer。在介绍native_setup之前，请注意一般都是在静态代码块中加载.so文件的，在MediaPlayer中有一段静态代码块，用于加载和链接库文件media_jni.so，早于构造函数，在加载类时就执行。一般全局性的数据、变量都可以放在这里。下面是加载和链接media_jni.so文件的代码：

```java
static {
        System.loadLibrary("media_jni");
        native_init();
    }
```

下面开始进入android_media_MediaPlayer.cpp分析，第一个函数android_media_Media-Player_native_init就是从Java静态代码块调过来的native_init：

```c++
android_media_MediaPlayer_native_init(JNIEnv* env)
{
  //类的句柄  clazz = env->FindClass("android/media/MediaPlayer");  
	jclass clazz;
	class = env->FindClass("android/media/MediaPlayer");
	if(clazz == NULL){//找不到，直接return
        return;
   }
	fields.context = env->GetFieldID(clazz, "mNativeContext");	// Java类中保存JNI层的mediaplayer对象
	
	/* JNI 事件通知Java，static 函数 */
	fields.post_event = env->GetStaticMethodID(clazz, "postEventFromNative", "(Ljava/lang/Object;IIILjava/lang/Object;)V");
 
	fields.surface_texture = env->GetFieldID(clazz, "mNativeSurfaceTexture", "I");
	
	jclass surface = env->FindClass("android/view/Surface");
	
	fields.bitmapClazz = env->FindClass("android/graphics/Bitmap");
	
	fields.bitmapContstructor = env->GetMethodID(fields.bitmapClazz, "<init>", "(I[BZ[BI)V");	// 找到Bitmap的构造函数
}
```

上面这种方式是通过JNI调用Java层的MediaPlayer类，然后拿到mNativeContext的指针，接着调用了MediaPlayer.java中的静态方法postEventFromNative，把Native的事件回调到Java层，使用EventHandler post事件回到主线程中，用软引用指向原生的MediaPlayer，以保证Native代码是安全的。代码如下：

```c++
private static void postEventFromNative(Object mediaplayer_ref, int what, int arg1, int arg2, Object obj)
{
	MediaPlayer mp = (MediaPlayer)((WeakReference)mediaplayer_ref).get();
  //省略部分代码...
	Message m = mp.mEventHandler.obtainMessage(what, arg1, arg2, obj);
	mp.mEventHandler.sendMessage(m);
}
//最后是通过EventHandler来处理。
```

之前我们在Java层的MediaPlayer.java文件的构造函数中，分析到最后有一个native_setup，在android_media_MediaPlayer.cpp中找到对应的函数，代码如下：

```c++
static void
android_media_MediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    ALOGV("native_setup");
  	//创建MediaPlayer c++对象
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
```

最后调用setMediaPlayer方法，代码如下：

```c++
static sp<MediaPlayer> setMediaPlayer(JNIEnv* env, jobject thiz, const sp<MediaPlayer>& player)
{
    Mutex::Autolock l(sLock);
  	
    sp<MediaPlayer> old = (MediaPlayer*)env->GetLongField(thiz, fields.context);
    if (player.get()) {
        player->incStrong((void*)setMediaPlayer);
    }
    if (old != 0) {
        old->decStrong((void*)setMediaPlayer);
    }
  	
    env->SetLongField(thiz, fields.context, (jlong)player.get());
    return old;
}
```

> 这块代码还没读懂，先放在这记录一下。

### 2.3、setDataSource过程

直接上setDataSource方法代码：

```java
private void setDataSource(String path, Map<String, String> headers, List<HttpCookie> cookies)
        throws IOException, IllegalArgumentException, SecurityException, IllegalStateException
{
    String[] keys = null;
    String[] values = null;
		
  	//取header信息，key、value分别放到两个数组中
    if (headers != null) {
        keys = new String[headers.size()];
        values = new String[headers.size()];

        int i = 0;
        for (Map.Entry<String, String> entry: headers.entrySet()) {
            keys[i] = entry.getKey();
            values[i] = entry.getValue();
            ++i;
        }
    }
  	//调用重载方法
    setDataSource(path, keys, values, cookies);
}
```

这里主要是取了header信息，key、value分别放到两个数组中，又调用了setDataSource的重载方法，代码如下：

```java
private void setDataSource(String path, String[] keys, String[] values,
            List<HttpCookie> cookies)
            throws IOException, IllegalArgumentException, SecurityException, IllegalStateException {
        final Uri uri = Uri.parse(path);
        final String scheme = uri.getScheme();
        if ("file".equals(scheme)) {//文件资源
            path = uri.getPath();
        } else if (scheme != null) {//非文件资源
            // handle non-file sources
            nativeSetDataSource(
                MediaHTTPService.createHttpServiceBinderIfNecessary(path, cookies),
                path,
                keys,
                values);
            return;
        }

        final File file = new File(path);
        try (FileInputStream is = new FileInputStream(file)) {
            setDataSource(is.getFD());//拿着文件标识符又调用了重载方法
        }
    }
```

这里转接调用重载方法，最终调用了native方法，开始进入JNI层，发现找不到android_media_MediaPlayer_setDataSource函数，但发现有一个函数名映射函数声明，这是JNI中常用的动态注册方法，代码如下：

```c++
static const JNINativeMethod gMethods[] = {
    {
        "nativeSetDataSource",
        "(Landroid/os/IBinder;Ljava/lang/String;[Ljava/lang/String;"
        "[Ljava/lang/String;)V",
        (void *)android_media_MediaPlayer_setDataSourceAndHeaders
    },

    {"_setDataSource",      "(Ljava/io/FileDescriptor;JJ)V",    (void *)android_media_MediaPlayer_setDataSourceFD},
    {"_setDataSource",      "(Landroid/media/MediaDataSource;)V",(void *)android_media_MediaPlayer_setDataSourceCallback },
    {"_setVideoSurface",    "(Landroid/view/Surface;)V",        (void *)android_media_MediaPlayer_setVideoSurface},
};
```

对以上这个函数名映射，如果看过JNIEnv * 源码的话，应该不会感到陌生，无非还是映射，不影响我们的分析。在这里接下来对android_media_MediaPlayer_setDataSourceFD函数进行分析：

```c++
static void
android_media_MediaPlayer_setDataSourceFD(JNIEnv *env, jobject thiz, jobject fileDescriptor, jlong offset, jlong length)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }

    if (fileDescriptor == NULL) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return;
    }
  	//在JNI中获取java.io.FileDescriptor
    int fd = jniGetFDFromFileDescriptor(env, fileDescriptor);
    ALOGV("setDataSourceFD: fd %d", fd);
    process_media_player_call( env, thiz, mp->setDataSource(fd, offset, length), "java/io/IOException", "setDataSourceFD failed." );
}
```

接着分析process_media_player_call函数：

```c++
// If exception is NULL and opStatus is not OK, this method sends an error
// event to the client application; otherwise, if exception is not NULL and
// opStatus is not OK, this method throws the given exception to the client
// application.
//如果异常为NULL且opStatus不正确，则此方法向客户端应用程序发送错误
//事件；否则，如果exception不为NULL并且
// opStatus不正确，则此方法将给定的异常抛出给客户端应用程序。
static void process_media_player_call(JNIEnv *env, jobject thiz, status_t opStatus, const char* exception, const char *message)
{
    if (exception == NULL) {  // Don't throw exception. Instead, send an event.
        if (opStatus != (status_t) OK) {
            sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
            if (mp != 0) mp->notify(MEDIA_ERROR, opStatus, 0);
        }
    } else {  // Throw exception!
        if ( opStatus == (status_t) INVALID_OPERATION ) {
            jniThrowException(env, "java/lang/IllegalStateException", NULL);
        } else if ( opStatus == (status_t) BAD_VALUE ) {
            jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        } else if ( opStatus == (status_t) PERMISSION_DENIED ) {
            jniThrowException(env, "java/lang/SecurityException", NULL);
        } else if ( opStatus != (status_t) OK ) {
            if (strlen(message) > 230) {
               // if the message is too long, don't bother displaying the status code
               jniThrowException( env, exception, message);
            } else {
               char msg[256];
                // append the status code to the message
               sprintf(msg, "%s: status=0x%X", message, opStatus);
               jniThrowException( env, exception, msg);
            }
        }
    }
}
```

总结以上代码：当mp->setDataSource(fd, offset, length)函数得到状态后，对各种状态进行通知。有异常的直接抛出，这样也就不会影响MediaPlayer后面的执行过程了。

接下来看看以HTTP/RTSP传入JNI。在Java层中对应的nativeSetDataSource函数如下：

```java
private native void nativeSetDataSource(
        IBinder httpServiceBinder, String path, String[] keys, String[] values)
        throws IOException, IllegalArgumentException, SecurityException, IllegalStateException;
```

在JNI中通过映射表可对应到android_media_MediaPlayer_setDataSourceAndHeaders函数：

```c++
static void
android_media_MediaPlayer_setDataSourceAndHeaders(
        JNIEnv *env, jobject thiz, jobject httpServiceBinderObj, jstring path,
        jobjectArray keys, jobjectArray values) {

    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }

    if (path == NULL) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return;
    }

    const char *tmp = env->GetStringUTFChars(path, NULL);
    if (tmp == NULL) {  // Out of memory
        return;
    }
    ALOGV("setDataSource: path %s", tmp);

    String8 pathStr(tmp);
    env->ReleaseStringUTFChars(path, tmp);
    tmp = NULL;

    // We build a KeyedVector out of the key and val arrays
    KeyedVector<String8, String8> headersVector;
    if (!ConvertKeyValueArraysToKeyedVector(
            env, keys, values, &headersVector)) {
        return;
    }

    sp<IMediaHTTPService> httpService;
    if (httpServiceBinderObj != NULL) {
      	//通过Binder将httpServiceBinderObj传给IPC并返回给binder
        sp<IBinder> binder = ibinderForJavaObject(env, httpServiceBinderObj);
      	//强制转换成IMediaHTTPService
        httpService = interface_cast<IMediaHTTPService>(binder);
    }
		//开始判断状态，和上面的文件操作是一样的
    status_t opStatus =
        mp->setDataSource(
                httpService,
                pathStr,
                headersVector.size() > 0? &headersVector : NULL);
		//见上面的文件操作
    process_media_player_call(
            env, thiz, opStatus, "java/io/IOException",
            "setDataSource failed." );
}
```

至此，setDataSource过程就完成了。这里需要注意两点：

1. 从Java→JNI→C++的正向调用过程（前面从Java层到Native层都是正向过程）。
2. C++→JNI→Java的过程（如mp->setDataSource( httpService, pathStr, headersVector.size() > 0? &headersVector : NULL）。

那有人肯定会问，这样来回调的好处是什么？好处有如下这几点：

-  安全性，封装在Native层的代码是so形式的，破坏性风险小。
- 效率高，在运行速度上C++执行时间短，且底层也是用C++语言编写的。对于复杂的渲染及对时间要求高的渲染，放在Native层是最好不过的选择。
- 连通性，正向调用将值传入，反向调用把处理过的值通知回去。相当于一根管道。

### 2.4、setDisPlay过程

接下来看看在setDataSource之后，开始进行的mp.setDisplay(holder)：

```java
public void setDisplay(SurfaceHolder sh) {
  			//1、设置一个控制器，用于展示视频图像
        mSurfaceHolder = sh;
        Surface surface;
        if (sh != null) {
            surface = sh.getSurface();
        } else {
            surface = null;
        }
  			//2、给视频设置Surface，带_的函数是native函数
        _setVideoSurface(surface);
  			//3、更新Surface到屏幕上
        updateSurfaceScreenOn();
    }
```

对于上面代码中的第2点，同样在android_media_MediaPlayer.cpp中找到其对应的函数：

```c++
static void
android_media_MediaPlayer_setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface)
{
    setVideoSurface(env, thiz, jsurface, true /* mediaPlayerMustBeAlive */);
}

static void
 setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface, jboolean mediaPlayerMustBeAlive)
 {
     sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
     if (mp == NULL) {
         if (mediaPlayerMustBeAlive) {
             jniThrowException(env, "java/lang/IllegalStateException", NULL);
         }
         return;
     }

     decVideoSurfaceRef(env, thiz);

     sp<IGraphicBufferProducer> new_st;
     if (jsurface) {
       		//得到java层的Surface
         sp<Surface> surface(android_view_Surface_getSurface(env, jsurface));
         if (surface != NULL) {
             new_st = surface->getIGraphicBufferProducer();
             if (new_st == NULL) {
                 jniThrowException(env, "java/lang/IllegalArgumentException",
                     "The surface does not have a binding SurfaceTexture!");
                 return;
             }
             new_st->incStrong((void*)decVideoSurfaceRef);
         } else {
             jniThrowException(env, "java/lang/IllegalArgumentException",
                     "The surface has been released");
             return;
         }
     }

     env->SetLongField(thiz, fields.surface_texture, (jlong)new_st.get());

     // This will fail if the media player has not been initialized yet. This
     // can be the case if setDisplay() on MediaPlayer.java has been called
     // before setDataSource(). The redundant call to setVideoSurfaceTexture()
     // in prepare/prepareAsync covers for this case.
     mp->setVideoSurfaceTexture(new_st);
 }

static void
decVideoSurfaceRef(JNIEnv *env, jobject thiz)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL) {
        return;
    }
		//得到旧的SurfaceTexture
    sp<IGraphicBufferProducer> old_st = getVideoSurfaceTexture(env, thiz);
    if (old_st != NULL) {
        old_st->decStrong((void*)decVideoSurfaceRef);
    }
}
```

这里有如下几个概念需要理解：

#### SurfaceTexture

SurfaceTexture是Android 3.0（API 11）加入的一个类。这个类跟SurfaceView很像，可以从视频解码里面获取图像流（image stream）。但是，和SurfaceView不同的是，SurfaceTexture在接收图像流之后，不需要显示出来。SurfaceTexture不需要显示到屏幕上，因此我们可以用SurfaceTexture接收解码出来的图像流，然后从SurfaceTexture中取得图像帧的副本进行处理，处理完毕后再送给另一个SurfaceView用于显示。

#### Surface

处理被屏幕排序的原生的Buffer，Android中的Surface就是一个用来画图形（graphic）或图像（image）的地方。对于View及其子类，都是画在Surface上的，各Surface对象通过SurfaceFlinger合成到frameBuffer。每个Surface都是双缓冲的（实际上就是两个线程，一个渲染线程，一个UI更新线程），它有一个backBuffer和一个frontBuffer。在Surface中创建的Canvas对象，可用来管理Surface绘图操作，Canvas对应Bitmap，存储Surface中的内容。

#### SurfaceView

在Camera、MediaRecorder、MediaPlayer中SurfaceView经常被用来显示图像。SurfaceView是View的子类，实现了Parcelable接口，其中内嵌了一个专门用于绘制的Surface，SurfaceView可以控制这个Surface的格式和尺寸，以及Surface的绘制位置。可以理解Surface就是管理数据的地方，SurfaceView就是展示数据的地方。

#### SurfaceHolder

顾名思义，是一个管理SurfaceHolder的容器。SurfaceHolder是一个接口，其可被理解为一个Surface的监听器。通过回调函数addCallback(SurfaceHolder.Callback callback)监听Surface的创建，通过获取Surface中的Canvas对象，锁定之。所得到的Canvas对象在完成修改Surface中的数据后，释放同步锁，并提交改变Surface的状态及图像，展示新的图像数据。

最后总结一下，SurfaceView中调用getHolder函数，可以获得当前SurfaceView中的Surface对应的SurfaceHolder，SurfaceHolder开始对Surface进行管理操作。这里按MVC模式可以更好地理解M:Surface（图像数据）、V:SurfaceView（图像展示）、C:SurfaceHolder（图像数据管理）。MediaPlayer.java中的setDisplay操作就是对将要显示的视频进行预设置。

以上就是setDisplay的过程，Java层中setDisplay的最后一行，就是通过JNI返回的Surface，时时做好更新准备。

## 三、开始prepare后的流程

前面分析了MediaPlayer从创建到setDataSource的过程，尽管分析了代码，但是没有从MediaPlayer生态上认识各类库之间的依赖调用关系。

MediaPlayer部分的头文件在frameworks/base/include/media/目录中，这个目录和libmedia.so库源文件的目录frameworks/base/media/libmedia/相对应。主要的头文件有以下几个。

-  IMediaPlayerClient.h
-  mediaplayer.h
-  IMediaPlayer.h
- IMediaPlayerService.h
-  MediaPlayerInterface.h
  在这些头文件中，mediaplayer.h提供了对上层的接口，而其他的几个头文件提供的是一些接口类（即包含了纯虚函数的类），这些接口类必须被实现类继承才能够使用。
  MediaPlayer各个具体类之间的依赖关系图如图2-3所示。

![image-20201126164746173](../../../Library/Application%20Support/typora-user-images/image-20201126164746173.png)

<center>图2-3 MediaPlayer各个具体类之间的依赖关系图</center>

在运行的时候，整个MediaPlayer可以大致上分成Client和Server两个部分，它们分别在两个进程中运行，它们之间使用Binder机制实现IPC通信。从框架结构上看，IMediaPlayer Service.h、IMediaPlayerClient.h和mediaplayer.h这3个头文件中定义了MediaPlayer的接口和架构，在目录中有专门的MediaPlayerService.cpp和mediaplayer.cpp文件对应上面3个头文件，用于MediaPlayer架构的实现。

在给播放器设置数据源且展现了Surface后，你应当开始调用prepare或prepareAsync函数。对于文件类型，调用prepare函数将暂时阻塞，因为prepare是一个同步函数，直到MediaPlayer已经准备好数据即将播放，也就是播放器回调了onPrepared函数，进入Prepared状态。

prepare函数的执行过程如下：

```java
/**
     * Prepares the player for playback, synchronously.
     *
     * After setting the datasource and the display surface, you need to either
     * call prepare() or prepareAsync(). For files, it is OK to call prepare(),
     * which blocks until MediaPlayer is ready for playback.
     *
     * @throws IllegalStateException if it is called in an invalid state
     */
    public void prepare() throws IOException, IllegalStateException {
        _prepare();
        scanInternalSubtitleTracks();

        // DrmInfo, if any, has been resolved by now.
        synchronized (mDrmLock) {
            mDrmInfoResolved = true;
        }
    }
```

Native层的android_media_MediaPlayer_prepare函数：

```c++
static void
android_media_MediaPlayer_prepare(JNIEnv *env, jobject thiz)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }

    // Handle the case where the display surface was set before the mp was
    // initialized. We try again to make it stick.
    sp<IGraphicBufferProducer> st = getVideoSurfaceTexture(env, thiz);//1
    mp->setVideoSurfaceTexture(st);//2

    process_media_player_call( env, thiz, mp->prepare(), "java/io/IOException", "Prepare failed." );//3
}

static sp<IGraphicBufferProducer>
getVideoSurfaceTexture(JNIEnv* env, jobject thiz) {
    IGraphicBufferProducer * const p = (IGraphicBufferProducer*)env->GetLongField(thiz, fields.surface_texture);
    return sp<IGraphicBufferProducer>(p);
}
```

我们曾经介绍过上面代码中的1、2、3，1中的getVideoSurfaceTexture获取一个IGraphicBufferProducer类型指针，2中是setVideoSurfaceTexture，主要给MediaPlayer传入IGraphicBufferProducer。这里IGraphicBufferProducer就是App和BufferQueue的重要桥梁，GraphicBufferProducer承担着单个应用进程中的UI显示需求，与BufferQueue打交道的就是它。

BpGraphicBufferProducer是GraphicBufferProducer在客户端这边的代理对象，负责和SurfaceFlinger交互，GraphicBufferProducer通过gbp（IGraphicBufferProducer类对象）向BufferQueue获取Buffer，然后填充UI信息，填充完毕会通知SurfaceFlinger。

3中是一个判定并且进行通知的函数，这个process_media_player_call是对MediaPlayer调用prepare函数后是否有异常的检测，如果出现参数不合法，或是I/O异常，就会抛出异常。

下面再来看看prepareAsync：

```java
public native void prepareAsync() throws IllegalStateException;
```

这是一个native函数声明，下面分析android_media_MediaPlayer_prepareAsync函数：

```c++
static void
android_media_MediaPlayer_prepareAsync(JNIEnv *env, jobject thiz)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }

    // Handle the case where the display surface was set before the mp was
    // initialized. We try again to make it stick.
    sp<IGraphicBufferProducer> st = getVideoSurfaceTexture(env, thiz);
    mp->setVideoSurfaceTexture(st);

    process_media_player_call( env, thiz, mp->prepareAsync(), "java/io/IOException", "Prepare Async failed." );
}
```

从代码来分析，除了最后process_media_player_call中的mp->prepareAsync在判断状态时不一样，其他都和prepare函数是一样的。它的操作结果经过回调通知Java层。
下面看看media/mediaplayer.h中的prepareAsync函数（C++代码）：

```c++
status_t MediaPlayer::prepareAsync()//之前所说的status_t就是从这里返回的
{
    ALOGV("prepareAsync");
    Mutex::Autolock _l(mLock);//这是一个互斥锁
    return prepareAsync_l();
}
```

接着分析prepareAsync_l函数实现代码：

```c++
// must call with lock held
//必须在锁住后调用此函数
status_t MediaPlayer::prepareAsync_l()
{
    if ( (mPlayer != 0) && ( mCurrentState & (MEDIA_PLAYER_INITIALIZED | MEDIA_PLAYER_STOPPED) ) ) {
        if (mAudioAttributesParcel != NULL) {
          	//设置音频流类型，在IMediaPlayer.cpp中对应的transact操作是
          	//SET_AUDIO_STREAM_TYPE
            mPlayer->setParameter(KEY_PARAMETER_AUDIO_ATTRIBUTES, *mAudioAttributesParcel);
        } else {
            mPlayer->setAudioStreamType(mStreamType);
        }
        mCurrentState = MEDIA_PLAYER_PREPARING;
        return mPlayer->prepareAsync();
    }
    ALOGE("prepareAsync called in state %d, mPlayer(%p)", mCurrentState, mPlayer.get());
    return INVALID_OPERATION;
}
```

下面继续分析prepareAsync函数，mp->prepareAsync对应的BnMediaPlayer操作如下：

```c++
MediaPlayerService::Client::prepareAsync
case PREPARE_ASYNC: {
		CHECK_INTERFACE(IMediaPlayer, data,replay);
  	reply->writeInt32(prepareAsync());
  	return NO_ERROR;
} break;
```

接着分析MediaPlayerService::Client::prepareAsync函数：

```c++
status_t MediaPlayerService::Client::prepareAsync()
{
  	sp<MeidaPlayerBase> p = getPlayer();
  	if(p == 0) return UNKNOW_ERROR;
  	status_t ret = p->prepareAsync();
  	#if CALLBACK_ANTAGONIZER
  	ALOGD("start Antagonizer");
  	if(ret == NO_ERROR) mAntagonizer->start();
  	#endif
  	return ret;
}
```

这里调用了AwesomePlayer的prepareAsync函数：

```c++
status_t AwesomePlayer::prepareAsync() {
    ATRACE_CALL();
    Mutex::Autolock autoLock(mLock);//互斥锁

    if (mFlags & PREPARING) {
        return UNKNOWN_ERROR;  // async prepare already pending
    }

    mIsAsyncPrepare = true;
    return prepareAsync_l();
}
```

接着分析AwesomePlayer::prepareAsync_l函数：

```c++
status_t AwesomePlayer::prepareAsync_l() {
    if (mFlags & PREPARING) {
        return UNKNOWN_ERROR;  // async prepare already pending
    }

    if (!mQueueStarted) {//队列不是开始状态时，设置成开始状态
        mQueue.start();
        mQueueStarted = true;
    }
		//修改状态为PREPARING
    modifyFlags(PREPARING, SET);
    mAsyncPrepareEvent = new AwesomeEvent(
            this, &AwesomePlayer::onPrepareAsyncEvent);
		//将回调事件通过队列通知出去
    mQueue.postEvent(mAsyncPrepareEvent);

    return OK;
}
```

总结一下上面的代码，首先判断mFlags，此时不是PREPARING。接着启动mQueue（类TimedEventQueue）。之后修改mFlags的状态为PREPARING，表示现在正在准备处理文件的音视频流。然后实例化一个AwesomeEvent，放到之前启动的mQueue中进行通知。

队列中处理的结果就是调用AwesomePlayer::onPrepareAsyncEvent函数。后面的过程就是初始化解码器，将流解码出来，也能知道视频流的宽高等属性，然后处于Prepared状态，不再向下跟踪。prepare的流程就完成了。

接下来，再回到Java层中之前的prepare函数中的scanInternalSubtitleTracks函数：

```java
private void scanInternalSubtitleTracks() {
        setSubtitleAnchor();

        populateInbandTracks();

        if (mSubtitleController != null) {
            mSubtitleController.selectDefaultTrack();
        }
    }
```

这个函数用于扫描内嵌字幕并进行跟踪，接下来看看MediaPlayer中的start函数：

```java
public void start() throws IllegalStateException {
        //FIXME use lambda to pass startImpl to superclass
        final int delay = getStartDelayMs();
        if (delay == 0) {
            startImpl();
        } else {
            new Thread() {
                public void run() {
                    try {
                        Thread.sleep(delay);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    baseSetStartDelayMs(0);
                    try {
                        startImpl();
                    } catch (IllegalStateException e) {
                        // fail silently for a state exception when it is happening after
                        // a delayed start, as the player state could have changed between the
                        // call to start() and the execution of startImpl()
                    }
                }
            }.start();
        }
    }

private void startImpl() {
        baseStart();
        stayAwake(true);
        _start();
    }

private void stayAwake(boolean awake) {
        if (mWakeLock != null) {
            if (awake && !mWakeLock.isHeld()) {
                mWakeLock.acquire();
            } else if (!awake && mWakeLock.isHeld()) {
                mWakeLock.release();
            }
        }
        mStayAwake = awake;
  			//更新Surface
        updateSurfaceScreenOn();
    }
```

首先执行PowerManager pm = (PowerManager)getSystemService(Context.POWER_SERVICE);，通过Context.getSystemService函数获取PowerManager实例。然后通过PowerManager的newWakeLock(int flags, String tag)来生成WakeLock实例。int flags指示要获取哪种WakeLock，不同的锁对CPU、屏幕、键盘灯有不同的影响。获取WakeLock实例后通过acquire获取相应的锁，然后进行其他业务逻辑的操作，最后使用release释放（释放是必需的）。

关于int flags，各种锁的类型对CPU、屏幕、键盘的影响如下。

- PARTIAL_WAKE_LOCK：保持CPU运转，屏幕和键盘灯有可能是关闭的。
- SCREEN_DIM_WAKE_LOCK：保持CPU运转，允许保持屏幕显示但有可能是灰的，允许关闭键盘灯。
- SCREEN_BRIGHT_WAKE_LOCK：保持CPU运转，允许保持屏幕高亮显示，允许关闭键盘灯。
- FULL_WAKE_LOCK：保持CPU运转，保持屏幕高亮显示，键盘灯也保持亮度。
- ACQUIRE_CAUSES_WAKEUP：正常唤醒锁实际上并不打开照明。相反，一旦打开，它们会一直保持。当获得WakeLock时，这个标志会使屏幕或/和键盘立即打开。一个典型的应用就是，可以立即看到对用户来说重要的通知。

最后，通过updateSurfaceScreenOn函数更新屏幕上的Surface。下面还是回到最上面的start函数中。在JNI中，对应的是android_media_MediaPlayer_start函数，代码如下：

```c++
static void
android_media_MediaPlayer_start(JNIEnv *env, jobject thiz)
{
    ALOGV("start");
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }
    process_media_player_call( env, thiz, mp->start(), NULL, NULL );
}
```

从MediaPlayer调用start函数开始，就进入了视频播放环节，最终会到C++的mediaplayer.cpp中实现，我们先分析一下mediaplayer.h（路径为\frameworks\av\include\media\mediaplayer.h）：

```c++
class MediaPlayer : public BnMediaPlayerClient,
                    public virtual IMediaDeathNotifier
{
public:
    MediaPlayer();
    ~MediaPlayer();
            void            died();
            void            disconnect();

            status_t        setDataSource(
                    const sp<IMediaHTTPService> &httpService,
                    const char *url,
                    const KeyedVector<String8, String8> *headers);

            status_t        setDataSource(int fd, int64_t offset, int64_t length);
            status_t        setDataSource(const sp<IDataSource> &source);
            status_t        setVideoSurfaceTexture(
                                    const sp<IGraphicBufferProducer>& bufferProducer);
            status_t        setListener(const sp<MediaPlayerListener>& listener);
            status_t        getBufferingSettings(BufferingSettings* buffering /* nonnull */);
            status_t        setBufferingSettings(const BufferingSettings& buffering);
            status_t        prepare();
            status_t        prepareAsync();
            status_t        start();
            status_t        stop();
            status_t        pause();
            bool            isPlaying();
            status_t        setPlaybackSettings(const AudioPlaybackRate& rate);
            status_t        getPlaybackSettings(AudioPlaybackRate* rate /* nonnull */);
            status_t        setSyncSettings(const AVSyncSettings& sync, float videoFpsHint);
            status_t        getSyncSettings(
                                    AVSyncSettings* sync /* nonnull */,
                                    float* videoFps /* nonnull */);
            status_t        getVideoWidth(int *w);
            status_t        getVideoHeight(int *h);
            status_t        seekTo(
                    int msec,
                    MediaPlayerSeekMode mode = MediaPlayerSeekMode::SEEK_PREVIOUS_SYNC);
            status_t        notifyAt(int64_t mediaTimeUs);
            status_t        getCurrentPosition(int *msec);
            status_t        getDuration(int *msec);
            status_t        reset();
            status_t        setAudioStreamType(audio_stream_type_t type);
            status_t        getAudioStreamType(audio_stream_type_t *type);
            status_t        setLooping(int loop);
            bool            isLooping();
            status_t        setVolume(float leftVolume, float rightVolume);
            void            notify(int msg, int ext1, int ext2, const Parcel *obj = NULL);
            status_t        invoke(const Parcel& request, Parcel *reply);
            status_t        setMetadataFilter(const Parcel& filter);
            status_t        getMetadata(bool update_only, bool apply_filter, Parcel *metadata);
            status_t        setAudioSessionId(audio_session_t sessionId);
            audio_session_t getAudioSessionId();
            status_t        setAuxEffectSendLevel(float level);
            status_t        attachAuxEffect(int effectId);
            status_t        setParameter(int key, const Parcel& request);
            status_t        getParameter(int key, Parcel* reply);
            status_t        setRetransmitEndpoint(const char* addrString, uint16_t port);
            status_t        setNextMediaPlayer(const sp<MediaPlayer>& player);

            media::VolumeShaper::Status applyVolumeShaper(
                                    const sp<media::VolumeShaper::Configuration>& configuration,
                                    const sp<media::VolumeShaper::Operation>& operation);
            sp<media::VolumeShaper::State> getVolumeShaperState(int id);
            // Modular DRM
            status_t        prepareDrm(const uint8_t uuid[16], const Vector<uint8_t>& drmSessionId);
            status_t        releaseDrm();
            // AudioRouting
            status_t        setOutputDevice(audio_port_handle_t deviceId);
            audio_port_handle_t getRoutedDeviceId();
            status_t        enableAudioDeviceCallback(bool enabled);
```

从接口中可以看出MediaPlayer类实现了一个MediaPlayer的基本播放控制操作，如播放（start）、停止（stop）、暂停（pause）、重置（reset）等。

另外的一个类DeathNotifier是在MediaPlayer类中定义的，它继承了IBinder类中的DeathRecipient类，这些类都是为进程间通信做准备的。

对于MediaPlayerClient和MediaPlayerService通过IPC进行通信。

可以发现调用start函数后，底层返回了一个状态，以便我们知道已经处于Started状态还是没有处于Started状态。这时需要用process_media_player_call判定这个返回的状态，然后通知Java层中的回调事件。

接下来，再分析一下MediaPlayer的pause函数：

```java
public void pause() throws IllegalStateException {
        stayAwake(false);//把唤醒状态置为false
        _pause();//调用native代码
        basePause();
    }
```

在对应的JNI中找到android_media_MediaPlayer_pause函数：

```c++
static void
android_media_MediaPlayer_pause(JNIEnv *env, jobject thiz)
{
    ALOGV("pause");
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    if (mp == NULL ) {
        jniThrowException(env, "java/lang/IllegalStateException", NULL);
        return;
    }
    process_media_player_call( env, thiz, mp->pause(), NULL, NULL );
}
```

在对应MediaPlayer.cpp中的pause函数：

```c++
status_t MediaPlayer::pause()
{
    ALOGV("pause");
    Mutex::Autolock _l(mLock);//互斥锁
    if (mCurrentState & (MEDIA_PLAYER_PAUSED|MEDIA_PLAYER_PLAYBACK_COMPLETE))
        return NO_ERROR;
    if ((mPlayer != 0) && (mCurrentState & MEDIA_PLAYER_STARTED)) {
        status_t ret = mPlayer->pause();
        if (ret != NO_ERROR) {
            mCurrentState = MEDIA_PLAYER_STATE_ERROR;
        } else {
            mCurrentState = MEDIA_PLAYER_PAUSED;
        }
        return ret;
    }
    ALOGE("pause called in state %d, mPlayer(%p)", mCurrentState, mPlayer.get());
    return INVALID_OPERATION;
}
```

查看pause函数，可以看到和start函数的流程类似，也是通过mp->pause返回对应的状态，然后通知上层来暂停的。

## 四、C++中MediaPlayer的C/S架构

这里来分析Java层的一个函数在C++层MediaPlayer中的过程（路径为frameworks/av/media/libmedia/MediaPlayer.cpp）。

下面找一个我们熟悉的setDataSource函数来看看C（Client）/S（Server）模式的过程。setDataSource函数如下：

```c++
status_t MediaPlayer::setDataSource(int fd, int64_t offset, int64_t length)
{
    ALOGV("setDataSource(%d, %" PRId64 ", %" PRId64 ")", fd, offset, length);
    //首先赋值为一个未知错误的状态，就像boolean值事先声明为false一样
  	status_t err = UNKNOWN_ERROR;
  	//通过IMdiaPlayerService获取service端的MediaPlayerService
    const sp<IMediaPlayerService> service(getMediaPlayerService());
    if (service != 0) {//如果service不为空
      	//调用service的create函数
        sp<IMediaPlayer> player(service->create(this, mAudioSessionId));
        if ((NO_ERROR != doSetRetransmitEndpoint(player)) ||
            (NO_ERROR != player->setDataSource(fd, offset, length))) {
            player.clear();
        }
        err = attachNewPlayer(player);
    }
    return err;
}
```

对应看看MediaPlayerService.cpp中的create函数，MediaPlayerService.cpp在C++ 6.0源码中处于frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp中，代码如下：

```c++
sp<IMediaPlayer> MediaPlayerService::create(const sp<IMediaPlayerClient>& client,
        audio_session_t audioSessionId)
{
    pid_t pid = IPCThreadState::self()->getCallingPid();
    int32_t connId = android_atomic_inc(&mNextConnId);

    sp<Client> c = new Client(
            this, pid, connId, client, audioSessionId,
            IPCThreadState::self()->getCallingUid());

  	//表示校验调用方的uid，用来进行身份验证
    ALOGV("Create new client(%d) from pid %d, uid %d, ", connId, pid,
         IPCThreadState::self()->getCallingUid());
		//把构造的Client强引用对象赋值成弱引用对象
    wp<Client> w = c;
    {
        Mutex::Autolock lock(mLock);//互斥锁
      	//mClients声明为SortedVector< wp<Client> >
        mClients.add(w);
    }
    return c;
}
```

在new Client中，有一个IPCThreadState。在Android中ProcessState是客户端和服务器端公共的部分，作为Binder通信的基础。ProcessState是一个singleton类，每个进程只有一个对象，这个对象负责打开Binder驱动，建立线程池，让其进程里面的所有线程都能通过Binder通信。

与之相关的是IPCThreadState，每个线程都有一个IPCThreadState实例登记在Linux线程的上下文附属数据中，主要负责Binder的读取、写入和请求处理。IPCThreadState在构造的时候获取进程的ProcessState并记录在自己的成员变量mProcess中，通过mProcess可以获得Binder的句柄。IPCThreadState通过IPCThreadState::transact把data及handle等填充进binder_transaction_data，在两个进程间通信。

这里这个Client到底是什么？我们又得追踪一下，在frameworks/av/media/libmediaplayerservice/MediaPlayerService.h中，如下：

```c++
class Client : public BnMediaPlayer {
        // IMediaPlayer interface
        virtual void            disconnect();
        virtual status_t        setVideoSurfaceTexture(
                                        const sp<IGraphicBufferProducer>& bufferProducer);
        virtual status_t        setBufferingSettings(const BufferingSettings& buffering) override;
        virtual status_t        getBufferingSettings(
                                        BufferingSettings* buffering /* nonnull */) override;
        virtual status_t        prepareAsync();
        virtual status_t        start();
        virtual status_t        stop();
        virtual status_t        pause();
        virtual status_t        isPlaying(bool* state);
        virtual status_t        setPlaybackSettings(const AudioPlaybackRate& rate);
        virtual status_t        getPlaybackSettings(AudioPlaybackRate* rate /* nonnull */);
        virtual status_t        setSyncSettings(const AVSyncSettings& rate, float videoFpsHint);
        virtual status_t        getSyncSettings(AVSyncSettings* rate /* nonnull */,
                                                float* videoFps /* nonnull */);
        virtual status_t        seekTo(
                int msec,
                MediaPlayerSeekMode mode = MediaPlayerSeekMode::SEEK_PREVIOUS_SYNC);
        virtual status_t        getCurrentPosition(int* msec);
        virtual status_t        getDuration(int* msec);
        virtual status_t        reset();
        virtual status_t        notifyAt(int64_t mediaTimeUs);
        virtual status_t        setAudioStreamType(audio_stream_type_t type);
        virtual status_t        setLooping(int loop);
        virtual status_t        setVolume(float leftVolume, float rightVolume);
    }; // Client
```

总结一下上面的代码，Client类的继承关系为Client->BnMediaPlayer->IMediaPlayer。分析上面的代码可以看出，create函数构造了一个Client对象，并将此Client对象添加到MediaPlayerService类的全局列表mClients中，这是一个SortedVector，紧接着执行player->setDataSource(url,headers)，即Clients::setDataSource，因此在setDataSource中有如下语句：

```c++
sp<IMediaPlayer> player(service->create(this, mAudioSessionId));
```

等价于

```c++
sp<IMediaPlayer> player(newClient(**));
```

即player最终是用Client对象来初始化的，可以直接认为player==Client。
这时候问题来了，在C++中，这个Client及MediaPlayer又是什么关系呢？

-  Client是MediaPlayerService内部的一个类，我们从上面的代码已知，因为MediaPlayerService运行在服务器端，故Client也运行在服务器端。
-  Client在MediaPlayerService.h中，那接着看看MediaPlayerService中的实现，实现过程中调用过MediaPlayerService类的一些函数，同样回到setDataSource。代码如下：

```c++
status_t MediaPlayerService::Client::setDataSource(
        const sp<IMediaHTTPService> &httpService,
        const char *url,
        const KeyedVector<String8, String8> *headers)
{
    ALOGV("setDataSource(%s)", url);
    if (url == NULL)
        return UNKNOWN_ERROR;
		//这里匹配来自HTTP、HTTPS、RTSP的相关url,这些流是需要经过网络传输的，检查其是否设置了相应的权限，如果没有，返回PERMISSON_DENIED
    if ((strncmp(url, "http://", 7) == 0) ||
        (strncmp(url, "https://", 8) == 0) ||
        (strncmp(url, "rtsp://", 7) == 0)) {
        if (!checkPermission("android.permission.INTERNET")) {
            return PERMISSION_DENIED;
        }
    }
		//判断是否通过contentprovider提供的数据
    if (strncmp(url, "content://", 10) == 0) {
        // get a filedescriptor for the content Uri and
        // pass it to the setDataSource(fd) method

        String16 url16(url);
        int fd = android::openContentProviderFile(url16);
        if (fd < 0)
        {
            ALOGE("Couldn't open fd for %s", url);
            return UNKNOWN_ERROR;
        }
        status_t status = setDataSource(fd, 0, 0x7fffffffffLL); // this sets mStatus
        close(fd);
        return mStatus = status;
    } else {
        player_type playerType = MediaPlayerFactory::getPlayerType(this, url);
        sp<MediaPlayerBase> p = setDataSource_pre(playerType);
        if (p == NULL) {
            return NO_INIT;
        }

        return mStatus =
                setDataSource_post(
                p, p->setDataSource(httpService, url, headers));
    }
}
```

接下来重新看看MediaPlayer中头文件定义的函数声明，方便对比Client中的函数，以下代码在frameworks/av/include/media/mediaplayer.h中：

```c++
class MediaPlayer : public BnMediaPlayerClient,
                    public virtual IMediaDeathNotifier
{
public:
    MediaPlayer();
    ~MediaPlayer();
            void            died();
            void            disconnect();

            status_t        setDataSource(
                    const sp<IMediaHTTPService> &httpService,
                    const char *url,
                    const KeyedVector<String8, String8> *headers);

            status_t        setDataSource(int fd, int64_t offset, int64_t length);
            status_t        setDataSource(const sp<IDataSource> &source);
            status_t        setVideoSurfaceTexture(
                                    const sp<IGraphicBufferProducer>& bufferProducer);
            status_t        setListener(const sp<MediaPlayerListener>& listener);
            status_t        getBufferingSettings(BufferingSettings* buffering /* nonnull */);
            status_t        setBufferingSettings(const BufferingSettings& buffering);
            status_t        prepare();
            status_t        prepareAsync();
            status_t        start();
            status_t        stop();
            status_t        pause();
            bool            isPlaying();
            //省略部分代码
};
```

这里的函数和Client中的函数是一一对应的，两者通过Client的代理类联系在了一起：

```c++
status_t MediaPlayer::setDataSource(
        const sp<IMediaHTTPService> &httpService,
        const char *url, const KeyedVector<String8, String8> *headers)
{
    ALOGV("setDataSource(%s)", url);
    status_t err = BAD_VALUE;
    if (url != NULL) {
        const sp<IMediaPlayerService> service(getMediaPlayerService());
        if (service != 0) {
            sp<IMediaPlayer> player(service->create(this, mAudioSessionId));
            if ((NO_ERROR != doSetRetransmitEndpoint(player)) ||
                (NO_ERROR != player->setDataSource(httpService, url, headers))) {
                player.clear();
            }
            err = attachNewPlayer(player);
        }
    }
    return err;
}

status_t MediaPlayer::attachNewPlayer(const sp<IMediaPlayer>& player)
{
    status_t err = UNKNOWN_ERROR;
    sp<IMediaPlayer> p;
    { // scope for the lock
        Mutex::Autolock _l(mLock);

        if ( !( (mCurrentState & MEDIA_PLAYER_IDLE) ||
                (mCurrentState == MEDIA_PLAYER_STATE_ERROR ) ) ) {
            ALOGE("attachNewPlayer called in state %d", mCurrentState);
            return INVALID_OPERATION;
        }

        clear_l();
        p = mPlayer;
        mPlayer = player;
        if (player != 0) {
            mCurrentState = MEDIA_PLAYER_INITIALIZED;
            err = NO_ERROR;
        } else {
            ALOGE("Unable to create media player");
        }
    }

    if (p != 0) {
        p->disconnect();
    }

    return err;
}
```

上面的两个函数，一个是MediaPlayer的setDataSource，会调到attachNewPlayer函数，这个函数最终会调用服务器端Client对应的函数。到这里可能有读者会想，IMediaPlayer.h和mediaplayer.h的区别是什么？那么下面介绍一下IMediaPlayer.h、mediaplayer.h、ImediaPlayer-Client.h的区别。

- 从包结构上看：IMediaPlayer和IMediaPlayerClient.h都在frameworks/av/media/libmedia包中，而mediaplayer.h在/av/include/media包中（前面已有代码贴出）。
-  从功能上看：它们肩负的职责也不一样。
  这里贴出IMediaPlayer.h及IMediaPlayerClient.h的代码，IMediaPlayer.h位于frameworks/av/media/libmedia包中：

```c++
#ifndef ANDROID_IMEDIAPLAYER_H
#define ANDROID_IMEDIAPLAYER_H

#include <utils/RefBase.h>
#include <binder/IInterface.h>
#include <binder/Parcel.h>
#include <utils/KeyedVector.h>
#include <system/audio.h>

#include <media/MediaSource.h>
#include <media/VolumeShaper.h>

// Fwd decl to make sure everyone agrees that the scope of struct sockaddr_in is
// global, and not in android::
struct sockaddr_in;

namespace android {

class Parcel;
class Surface;
class IDataSource;
struct IStreamSource;
class IGraphicBufferProducer;
struct IMediaHTTPService;
struct AudioPlaybackRate;
struct AVSyncSettings;
struct BufferingSettings;

typedef MediaSource::ReadOptions::SeekMode MediaPlayerSeekMode;

class IMediaPlayer: public IInterface
{
public:
    DECLARE_META_INTERFACE(MediaPlayer);

    virtual void            disconnect() = 0;

    virtual status_t        setDataSource(
            const sp<IMediaHTTPService> &httpService,
            const char *url,
            const KeyedVector<String8, String8>* headers) = 0;

    virtual status_t        setDataSource(int fd, int64_t offset, int64_t length) = 0;
    virtual status_t        setDataSource(const sp<IStreamSource>& source) = 0;
    virtual status_t        setDataSource(const sp<IDataSource>& source) = 0;
    virtual status_t        setVideoSurfaceTexture(
                                    const sp<IGraphicBufferProducer>& bufferProducer) = 0;
    virtual status_t        getBufferingSettings(
                                    BufferingSettings* buffering /* nonnull */) = 0;
    virtual status_t        setBufferingSettings(const BufferingSettings& buffering) = 0;
    virtual status_t        prepareAsync() = 0;
    virtual status_t        start() = 0;
    virtual status_t        stop() = 0;
    virtual status_t        pause() = 0;
    virtual status_t        isPlaying(bool* state) = 0;
    virtual status_t        setPlaybackSettings(const AudioPlaybackRate& rate) = 0;
    virtual status_t        getPlaybackSettings(AudioPlaybackRate* rate /* nonnull */) = 0;
    virtual status_t        setSyncSettings(const AVSyncSettings& sync, float videoFpsHint) = 0;
    virtual status_t        getSyncSettings(AVSyncSettings* sync /* nonnull */,
                                            float* videoFps /* nonnull */) = 0;
    virtual status_t        seekTo(
            int msec,
            MediaPlayerSeekMode mode = MediaPlayerSeekMode::SEEK_PREVIOUS_SYNC) = 0;
    virtual status_t        getCurrentPosition(int* msec) = 0;
    virtual status_t        getDuration(int* msec) = 0;
    virtual status_t        notifyAt(int64_t mediaTimeUs) = 0;
    virtual status_t        reset() = 0;
    virtual status_t        setAudioStreamType(audio_stream_type_t type) = 0;
    virtual status_t        setLooping(int loop) = 0;
    virtual status_t        setVolume(float leftVolume, float rightVolume) = 0;
    virtual status_t        setAuxEffectSendLevel(float level) = 0;
    virtual status_t        attachAuxEffect(int effectId) = 0;
    virtual status_t        setParameter(int key, const Parcel& request) = 0;
    virtual status_t        getParameter(int key, Parcel* reply) = 0;
    virtual status_t        setRetransmitEndpoint(const struct sockaddr_in* endpoint) = 0;
    virtual status_t        getRetransmitEndpoint(struct sockaddr_in* endpoint) = 0;
    virtual status_t        setNextPlayer(const sp<IMediaPlayer>& next) = 0;

    virtual media::VolumeShaper::Status applyVolumeShaper(
                                    const sp<media::VolumeShaper::Configuration>& configuration,
                                    const sp<media::VolumeShaper::Operation>& operation) = 0;
    virtual sp<media::VolumeShaper::State> getVolumeShaperState(int id) = 0;

    // Modular DRM
    virtual status_t        prepareDrm(const uint8_t uuid[16],
                                    const Vector<uint8_t>& drmSessionId) = 0;
    virtual status_t        releaseDrm() = 0;

    // Invoke a generic method on the player by using opaque parcels
    // for the request and reply.
    // @param request Parcel that must start with the media player
    // interface token.
    // @param[out] reply Parcel to hold the reply data. Cannot be null.
    // @return OK if the invocation was made successfully.
    virtual status_t        invoke(const Parcel& request, Parcel *reply) = 0;

    // Set a new metadata filter.
    // @param filter A set of allow and drop rules serialized in a Parcel.
    // @return OK if the invocation was made successfully.
    virtual status_t        setMetadataFilter(const Parcel& filter) = 0;

    // Retrieve a set of metadata.
    // @param update_only Include only the metadata that have changed
    //                    since the last invocation of getMetadata.
    //                    The set is built using the unfiltered
    //                    notifications the native player sent to the
    //                    MediaPlayerService during that period of
    //                    time. If false, all the metadatas are considered.
    // @param apply_filter If true, once the metadata set has been built based
    //                     on the value update_only, the current filter is
    //                     applied.
    // @param[out] metadata On exit contains a set (possibly empty) of metadata.
    //                      Valid only if the call returned OK.
    // @return OK if the invocation was made successfully.
    virtual status_t        getMetadata(bool update_only,
                                        bool apply_filter,
                                        Parcel *metadata) = 0;

    // AudioRouting
    virtual status_t        setOutputDevice(audio_port_handle_t deviceId) = 0;
    virtual status_t        getRoutedDeviceId(audio_port_handle_t *deviceId) = 0;
    virtual status_t        enableAudioDeviceCallback(bool enabled) = 0;
};

// ----------------------------------------------------------------------------

class BnMediaPlayer: public BnInterface<IMediaPlayer>
{
public:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
};

}; // namespace android

#endif // ANDROID_IMEDIAPLAYER_H
```

在IMediaPlayer.h中定义的基本上都是虚函数，而我们知道虚函数在C++中用于实现多态性（Polymorphism），多态性是将接口与具体实现代码进行了分离，用形象的语言来解释就是以共同的方法实现，但因个体差异而采用不同的策略。所以它的功能是实现MediaPlayer功能的接口，看到onTransact函数，自然联想Binder通信，把底层的Parcel指针类型数据向上层的另一个进程传递。

再分析一下IMediaPlayerClient.h，同样位于frameworks/av/media/libmedia包中：

```c++
#include <utils/RefBase.h>
#include <binder/IInterface.h>
#include <binder/Parcel.h>

#include <media/IMediaPlayerClient.h>

namespace android {

enum {
    NOTIFY = IBinder::FIRST_CALL_TRANSACTION,
};

class BpMediaPlayerClient: public BpInterface<IMediaPlayerClient>
{
public:
    explicit BpMediaPlayerClient(const sp<IBinder>& impl)
        : BpInterface<IMediaPlayerClient>(impl)
    {
    }

    virtual void notify(int msg, int ext1, int ext2, const Parcel *obj)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IMediaPlayerClient::getInterfaceDescriptor());
        data.writeInt32(msg);
        data.writeInt32(ext1);
        data.writeInt32(ext2);
        if (obj && obj->dataSize() > 0) {
            data.appendFrom(const_cast<Parcel *>(obj), 0, obj->dataSize());
        }
        remote()->transact(NOTIFY, data, &reply, IBinder::FLAG_ONEWAY);
    }
};

IMPLEMENT_META_INTERFACE(MediaPlayerClient, "android.media.IMediaPlayerClient");

// ----------------------------------------------------------------------

status_t BnMediaPlayerClient::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch (code) {
        case NOTIFY: {
            CHECK_INTERFACE(IMediaPlayerClient, data, reply);
            int msg = data.readInt32();
            int ext1 = data.readInt32();
            int ext2 = data.readInt32();
            Parcel obj;
            if (data.dataAvail() > 0) {
                obj.appendFrom(const_cast<Parcel *>(&data), data.dataPosition(), data.dataAvail());
            }

            notify(msg, ext1, ext2, &obj);
            return NO_ERROR;
        } break;
        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}

} // namespace android
```

总结一下上面的代码，在内部定义一个BpMediaPlayerClient类（也就是Client的父类），然后它有一个onTransact函数。一般onXXX都是被动回调过来的，不是由自己控制的，如Activity中的onCreate、onPause、onStart函数，这些函数都是在其他地方处理并通知到Activity中的。这里也是一样的，onTransact作为Binder通信中的回调函数，前面介绍到player实际上是C/S模式的，IMediaPlayerClient.h的功能是描述一个MediaPlayer客户端的接口。

综上所述，mediaplayer.h的功能是对外（JNI层）的接口类，它最主要的是定义了一个MediaPlayer类（C++层），我们在android_media_MediaPlayer.cpp中就引入了media/mediaplayer.h；IMediaPlayer.h则是一个实现MediaPlayer（C++层）功能的接口；而IMediaPlayerClient.h的功能是描述一个MediaPlayer客户端（这里暂且理解为前面说的Client）的接口。