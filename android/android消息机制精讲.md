这篇文章一起来学习几个问题：

1. Handler、Looper、MessageQueue之间的关系？
2. Handler.sendMessage(Message msg)做了什么？
3. Handler如何实现线程切换？
4. 主线程的Looper死循环为什么不会卡顿？（EPoll）
5. 什么是同步消息、异步消息、同步屏障？

## Handler、Looper、Message之间的关系？

​		下图这张图大致的说明了下它们三者之间的关系：

![image-20200924141208001](/Users/user/wLog/android/image-20200924141208001.png)

​		Handler有两个成员变量Looper和MessageQueue，其中MessageQueue也是Looper的成员变量。Handler中的MessageQueue其实是Looper中的MessageQueue，看下面的代码：

```java
public Handler(Callback callback, boolean async) {
    //...省略

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
  	//将Looper中的MessageQueue赋值给了Handler的MessageQueue
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

## Handler.sendMessage(Message msg)做了什么？

主要做了两个操作：

1. 消息插入队列
2. 消息出队列中取出执行

### 消息插入队列

sendMessage(Message msg)最终调用的是sendMessageAtTime(Message msg, long uptimeMillis)方法，如下：

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
  	//这里最终调用的MessageQueue里的enqueueMessage(msg, uptimeMillis)
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

直接看MessageQueue里的enqueueMessage(msg, uptimeMillis)方法:

```java
boolean enqueueMessage(Message msg, long when) {
    //...省略

    synchronized (this) {
        //...省略

        msg.markInUse();
      	//把消息的执行时间赋值给了msg
        msg.when = when;
      	//这里注意下，mMessages是消息队列的头
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
          	//队列头是空|插在队列前面的消息(这里后面那个方法会讲到)|执行时间比队列头消息要早
          	//就会把这条消息作为队列头
            msg.next = p;
            mMessages = msg;
          	//mBlocked表示当前线程是否处于阻塞状态，处于阻塞状态就需要唤醒。
            needWake = mBlocked;
        } else {
            //延迟消息按执行时间做排序，找相应位置插入
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {//唤醒当前线程
            nativeWake(mPtr);
        }
    }
    return true;
}
```

```java
//这里发送插入队列前面的消息（插入到队头）
public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}
```

这里就完成了消息插入列表的操作。下面接着看**消息队列中的消息是如何执行的**。

### 消息出队列中取出执行

消息执行是在Looper的loop方法中循环取消息去执行的。

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
  
		//...省略

    for (;;) {
      	//从消息队列中取消息
        Message msg = queue.next(); // might block（queue.next方法可能会阻塞线程，后面会讲这个方法）
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

      	//...省略
        try {
          	//执行消息
            msg.target.dispatchMessage(msg);
            //...省略
        } catch (Exception exception) {
            //...省略
        } finally {
            //...省略
        }
				//...省略
        msg.recycleUnchecked();
    }
}
```

从消息队列中取消息，从上面的代码里可以知道是在MessageQueue的next()方法里取。

取消息分为三种情况：

1. 队列头消息为Null，说明已经没有可执行消息了，这里时候线程会休眠，等待插入下一条消息的时候唤醒。（enqueueMessage方法最后会调用nativeWake()方法）
2. 队列头消息为同步屏障，这时会屏蔽队列里的同步消息，顺序取出第一条异步消息（同步屏障详解请参考：https://www.jianshu.com/p/28fba43ac0b0）
3. 队列头为同步消息或者是异步消息。

第2和第3的情况的话会去判断是否到执行时间，如果到了则返回消息，否则休眠。

这里休眠分为三种情况：

1. nextPollTimeoutMillis==-1，休眠，等待下一条消息插入时唤醒
2. nextPollTimeoutMillis==0，不休眠
3. nextPollTimeoutMillis==某个正整数，设置休眠时间，到时间唤醒。

如下：

```java
Message next() {
    //...省略
  
  	//休眠时间，如果值为-1，则需要等待有新消息进来才会唤醒
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
				//调用native休眠方法，根据参数执行不同的休眠策略
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
          
          	//这里target为空的时候，说明队列头消息为同步屏障
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
              	//这里同步屏障会屏蔽掉同步消息，取出异步消息执行。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
          	//走到这里已经取到了消息
            if (msg != null) {
              	//消息未到执行时间
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.		
           					//计算休眠时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                  	
                    if (prevMsg != null) {
                      	//消息在队列中间
                        prevMsg.next = msg.next;
                    } else {
                      	//消息在队列头
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
						//...省略
            

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

## Handler如何实现线程切换？

这个问题其实第一个问题已经回答了，在A线程里发消息，插入消息队列中，在Looper一直在自己所在的线程循环取消息执行，那么消息的执行自然就切换到了Looper所在线程里了。所以这里就解释了Looper为什么要线程唯一。

## 主线程的Looper死循环为什么不会卡顿？（EPoll）

取消息那里讲到了几种情况下线程会进入休眠状态，这时会让出cpu资源。

## 什么是同步消息、异步消息、同步屏障？

参考：https://www.jianshu.com/p/28fba43ac0b0