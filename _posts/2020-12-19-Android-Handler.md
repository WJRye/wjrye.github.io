---
layout: post
title: Android 中 Handler 相关的面试问题分析
categories: [Android]
description: Handler，Message，MessageQueue，Looper 组成了 Android 系统中的消息机制。
keywords: Android, Handler
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false## Kotlin Multiplatform

---

## 简要

关于Android 系统中Handler，Message，MessageQueue，Looper 组成的消息机制，由于它们是开发 Android 应用程序的基础，所以在面试过程中，一定会做考察。而且每个问题必追究其原理，本篇文章将剖析曾遇到过的问题。

## Handler 问题相关

Handler 在 Android 系统消息机制中的职责是：**负责发送和接收处理消息**。

### Handler 中的方法 sendMessage 和 post 的区别？

这里先看 方法 sendMessage 的源码：

```
   public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

     public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

      private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

再看方法 post  的源码：

```
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

从两个方法源码，可以看出：

- sendMessage &rarr; sendMessageDelayed &rarr; sendMessageAtTime &rarr; enqueueMessage &rarr; ...
- post &rarr; sendMessageDelayed &rarr; sendMessageAtTime &rarr; enqueueMessage &rarr; ...

不管是方法 sendMessage，还是方法 post，它们只是得到的消息实体对象 Message 不一样：sendMessage 是通过外部传入的，一般外部会通过 Handler 的 obtainMessage 去获取一个Message，然后给 Message 的 what，arg1，obj 变量等赋值；而 post 是通过内部创建一个Message，并给 Message 的 callback 变量赋值（post 方法的参数 Runnable）。

另外，正是由于 Message 对象不一样，导致了 Handler 在接收处理消息的时候的逻辑也会不一样：

```
    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }

    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

在 方法 dispatchMessage 中，从代码逻辑可以看出：通过方法 sendMessage  和 post 发送一个消息，在接收处理消息时，会优先交给经过方法 post 发送的（或 经过给 Message 的 callback赋值发送的），如果不是，再交给 mCallback 对象（在创建 Handler 对象时传递进来的），如果再不是，才交给方法 sendMessage 发送的。

所以接收处理消息优先级是：

1. 先交给通过 post 方法发送的；
2. 然后交给 创建 Handler 对象时传递进来的回调接口 Handler.Callback；
3. 最后交给通过 sendMessage 方法发送的。

#### 答案

对于这个面试问题，所以回答就是：**通过方法 sendMessage 和 post 发送一个消息，在发送时对消息实体对象 Message 的赋值不一样，sendMessage 通过外部获取一个 Message，post 通过内部创建一个 Message，并给它的 callback 属性赋值；在接收处理时，会优先处理经过方法 post 发送的消息，其次才会去处理经过方法 sendMessage 发送的消息**。

### Handler 是如何发送一个延时消息的？

发送一个延时消息，是通过 Handler 提供的方法`sendMessageDelayed` 和 `postDelayed` 来实现的。它们的方法调用链如下：

- sendMessageDelayed ：sendMessageDelayed &rarr; sendMessageAtTime &rarr; enqueueMessage &rarr; ...
- postDelayed：postDelayed &rarr; sendMessageDelayed &rarr; sendMessageAtTime &rarr; enqueueMessage &rarr; ...

这两个方法调用实质是一样的，所以看方法 sendMessageDelayed 的源码就行了：

```
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

   public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        //省略
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        //省略
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

外面传入的延迟时间 `delayMillis` ，经过 sendMessageDelayed 方法后，会通过`SystemClock.uptimeMillis() + delayMillis` 再传入 MessageQueue 的 方法 enqueueMessage 的参数 when：

```
    boolean enqueueMessage(Message msg, long when) {
        //省略...
        synchronized (this) {
            //省略.. 
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            //如果当前消息队列为空，或消息执行时间为0，或消息执行时间小于当前链表第一个元素的执行时间，就将当前消息放入链表的首部
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                //省略.. 
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                // 根据链表中每个消息的执行时间，也就是Message 的 when 变量的值对链表进行排序
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
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

在这个方法中：`msg.when = when;`，就是将外面的`SystemClock.uptimeMillis() + delayMillis` 的值给`msg.when`赋值，关于`msg.when`，它就是 Message 的执行时间。赋值之后后面的逻辑是：如果当前消息队列为空，或消息执行时间为0，或消息执行时间小于当前链表第一个元素的执行时间，就将当前消息放入链表的首部，然后退出该方法；否则，根据链表中每个消息的执行时间，也就是Message 的 when 变量的值对链表进行排序，然后退出该方法。

#### 答案

对于这个面试问题，所以回答就是：**发送一个延时消息，会将当前时间+延迟时间的值给 Message 的 when 变量赋值，when 变量就是消息的执行时间。另外，在将消息插入到 MessageQueue 中时，还会根据链表中每个消息的执行时间，也就是Message 的 when 变量的值对链表中的 Message 进行排序**。

### 使用 Handler 先发送一个延迟 10s 的消息，再发送一个延迟 5s 的消息，然后让主线程 sleep 5s，最后消息执行顺序？

从问题**Handler 是如何发送一个延时消息的**，知道消息最后的执行时间，也就是`msg.when`  的值，而`msg.when` 的值为：当前时间+延迟时间。当 Handler 发送一个延迟消息时，MessageQueue 会根据链表中每个消息的执行时间（`msg.when`），对 Message 进行排序。

#### 答案

对于这个面试问题，所以回答就是：**发送一个延迟 10s 和 5s 的消息，不管先后顺序，MessageQueue 在存储消息时，总会把延迟 5s 的消息放在延迟 10s 的消息前面。所以，等主线程 sleep 5s  完后，延迟 5s 的消息的会先执行，然后执行 延迟 10s 的消息。**

## MessageQueue 问题相关

Message在 Android 系统消息机制中的职责是：**负责封装消息实体，也就是相关传递的参数**。
MessageQueue 在 Android 系统消息机制中的职责是：**负责以链表的形式管理 Message**。

### 如何把一个消息放在 MessageQueue 的首部？

 从上面分析 MessageQueue 的 enqueueMessage 方法，知道 **如果当前消息队列为空，或消息执行时间为0，或消息执行时间小于当前链表第一个元素的执行时间，就将当前消息放入链表的首部**。对于这三种情况，接下来将逐步分析。

#### 队列为空

MessageQueue 提供了 `isIdle`  方法来判断 当前队列是否为为空：

```
Looper.myQueue().isIdle()
```

如果为空，直接发送一个消息就好。

#### 消息执行时间为0

Handler 使用方法  send 或  post 发送一个消息，最后都会调用方法 `sendMessageAtTime`， 而这个方法的参数 `uptimeMillis` 传递到 MessageQueue 的 enqueueMessage 方法中，就会给 `msg.when` 赋值，所以只需给`uptimeMillis`赋值为 0 即可：

```
handler.sendMessageAtTime(handler.obtainMessage(),0);
```

其实，Handler  中也提供了方法：`sendMessageAtFrontOfQueue`：

```
    public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0); // uptimeMillis=0
    }

    public final boolean postAtFrontOfQueue(@NonNull Runnable r) {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
```

也是给参数 `uptimeMillis` 赋值为0： `uptimeMillis=0`。

#### 消息执行时间小于当前链表第一个元素的执行时间

和 **消息执行时间为0** 一样：

```
handler.sendMessageAtTime(handler.obtainMessage(),Long.MIN_VALUE);
```

不过，没有必要去这样做，只要按照上**消息执行时间为0** 去做就行。

#### 答案

对于这个面试问题，所以回答就是：**在 MessageQueue 的 enqueueMessage 方法中，会有这样的判断：如果当前消息队列为空，或消息执行时间为0，或消息执行时间小于当前链表第一个元素的执行时间，就将当前消息放入链表的首部。所以，对当消息队列为空时，通过`Looper.myQueue().isIdle()`方法来判断，如果判断为空，直接发送一个消息就行；对当消息队列不为空时，通过调用`sendMessageAtFrontOfQueue`就行，其实就是将消息执行时间`msg.when`赋值为0。**

### 如何让  Handler 发送一个消息并让它优先执行？

首先，需要了解 Message 的分类：

1. 同步消息，也就是普通消息
2. 异步消息

在 Message 类中，字段 `flag`  标记了当前消息是同步消息还是异步消息：

```
    /** If set message is asynchronous */
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;

    @UnsupportedAppUsage
    /*package*/ int flags;

    /**
     * Returns true if the message is asynchronous, meaning that it is not
     * subject to {@link Looper} synchronization barriers.
     *
     * @return True if the message is asynchronous.
     *
     * @see #setAsynchronous(boolean)
     */
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }

    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
```

从上面看出，只需  `msg.setAsynchronous(true);` ，该消息就是异步消息，默认是同步消息。

通常，设置 Message 是同步还是异步是通过 Handler 来设置的。

下面看 类 Handler 中的源码：

```
// 该变量标记 Message 是同步的还是异步的，默认为 false
 final boolean mAsynchronous;

    public Handler() {
        this(null, false);
    }

    public Handler(@Nullable Callback callback) {
        this(callback, false);
    }

    public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }

    public Handler(@NonNull Looper looper, @Nullable Callback callback) {
        this(looper, callback, false);
    }

    /**
     * @hide
     */
    @UnsupportedAppUsage
    public Handler(boolean async) {
        this(null, async);
    }

    /**
     * @hide 隐藏方法，外部不可调用
     */
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    /**
     * @hide 隐藏方法，外部不可调用
     */
    @UnsupportedAppUsage
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    @NonNull
    public static Handler createAsync(@NonNull Looper looper) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        return new Handler(looper, null, true); //mAsynchronous = true
    }

    @NonNull
    public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        if (callback == null) throw new NullPointerException("callback must not be null");
        return new Handler(looper, callback, true); //mAsynchronous = true
    }

    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) { //mAsynchronous = true 时，  Message 会设置为异步的，否则都为同步的。
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

Handler  中的 `mAsynchronous` 变量，决定了 Message  是同步的还是异步的。通常，使用构造方法创建的 Handler 对象，对 Message 的处理都是同步的，也就是  `mAsynchronous = false`；使用 `createAsync` 创建的 Handler 对象，对  Message 的处理都是异步的，也就是`mAsynchronous = true`。最后，在方法 `enqueueMessage` 中来设置   `msg.setAsynchronous`  Message 是同步还是异步。

虽然按照上面去设置，但是 MessageQueue 对 不管是同步还是异步消息不会做特殊处理。默认发送的消息都是同步消息，异步消息只有在 MessageQueue 设置了**同步屏障**才会凸显其作用。

#### 同步屏障

同步屏障的作用：让异步消息优先执行。

MessageQueue 提供了方法 `postSyncBarrier` 来设置同步屏障：

```
    /**
     * @hide 隐藏方法，外部不可调用
     */
    @TestApi
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

从代码逻辑可以看出，设置同步屏障，只是往 消息队列中 添加了一个 `msg.target = null` 的消息。通常，使用 Handler 发送的消息，`msg.target` 是不可能为空，即使为空，在 MessageQueue 的 `enqueueMessage` 方法中也会校验抛出异常。

设置同步屏障后，异步消息在 MessageQueue 的 `next` 方法中的处理：

```
    @UnsupportedAppUsage
    Message next() {
        // 省略...
        for (;;) {
           // 省略...
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //msg.target=null，循环找到找到第一个异步消息才终止
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //此时，如果找到了异步消息，就把这个消息返回出去
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // 省略...
                }
                // 省略...
            }
            // 省略...
        }
    }
```

当`msg.target = null` 时，会首先从 消息队列中去寻找异步消息（也就是 `msg.isAsynchronous()` 返回 true），如果找到了异步消息，本次就会优先处理这个消息。所以**异步消息只有在设置了同步屏障下，才会发生作用，它将优先于同步消息（普通消息）执行**。

#### 答案

对于这个面试问题，所以回答就是：

1. 使用 Handler 的方法 `createAsync`  创建一个发送异步消息的 Handler；或通过 `msg.setAsynchronous(true);` 来设置这个消息是异步消息；
2. 使用 `  mHandler.getLooper().getQueue().postSyncBarrier();` 来设置同步屏障；
3. 发送 异步消息，这样 MessageQueue 就会优先处理这类消息。

#### View 的绘制过程中 Handler 发送的消息和普通 Handler 发送的消息有何异同？

这里先复习下 View  的绘制过程，从 ViewRootImpl 开始，依次：`scheduleTraversals` &rarr; `doTraversal` &rarr; `performTraversals()` &rarr; 

1. `performMeasure` &rarr;  ...
2. `performLayout` &rarr;  ...
3. `performDraw` &rarr;  ...

方法 `scheduleTraversals` 源码：

```
   int mTraversalBarrier;

    @UnsupportedAppUsage
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

    void unscheduleTraversals() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            mChoreographer.removeCallbacks(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        }
    }
```

可以看出，首先会设置同步屏障：`mHandler.getLooper().getQueue().postSyncBarrier()`。

其次，关于 mChoreographer  的 `postCallback` 方法：`postCallback` &rarr; `postCallbackDelayed` &rarr; `postCallbackDelayedInternal()` &rarr; 

```
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }

    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```

这里，重点就是 Handler 发送的 Message 都设置为了异步消息：`msg.setAsynchronous(true);`。这也保证了 **View 绘制相关发送的消息都会优先执行**。

## Looper 问题相关

Looper 在 Android 系统消息机制中的职责是：**负责从 MessageQueue 中取出 Message，并把 Message 交给 Handler 去处理**。

### 可以在子线程中使用 Handler 发送和处理消息吗？

Android 系统提供的类 HandlerThread ，就是在子线程中创建的 Handler :

```
public class HandlerThread extends Thread {
    @Override
    public void run() {
        mTid = Process.myTid();
        // 创建当前线程的 Looper
        Looper.prepare();
        synchronized (this) {
           // 获取当前线程的 Looper
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        // 开始消息循环
        Looper.loop();
        mTid = -1;
    }
}
```

在调用 Thread 的 start 方法后，就会执行它的 run 方法，上面在 run 方法中做的事情是：

1. `Looper.prepare()`：创建当前线程的 Looper；
2. `mLooper = Looper.myLooper()`：获取当前线程的 Looper；
3. `Looper.loop()`：开始消息循环。

这里的关键点在于类 Looper 的方法 `Looper.prepare()`：

```
       @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

通过 ThreadLocal 对象来存储 当前线程的 Looper 对象。

然后，方法 `Looper.myLooper()`：

```
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

通过 ThreadLocal 对象来获取当前线程的 Looper 对象。

所以  HanderThread  的使用方式是：

```
        HandlerThread handlerThread = new HandlerThread("HandlerThread");
        Handler handler = new Handler(handlerThread.getLooper());
        handler.sendMessage(Message.obtain(handler));
```

这样，就可以在子线程中使用 Handler 发送和处理消息了。

#### 答案

对于这个面试问题，所以回答就是：**在子线程中使用 Handler 发送和处理消息，需要在子线程中创建对应的 Looper 对象，并让这个 Looper 对象和 Handler 对象关联起来。 Android 系统的类 HandlerThread 就提供了这样的功能。**

#### 思考：ThreadLocal 的原理？

### 如何判断当前线程是主线程？

要回答这个问题，首先需要知道 ThreadLocal 的一些简要信息：

1. 线程范围内的数据共享，针对某一个线程存储数据；
2. 每一个线程 Thread 都有一个 `ThreadLocalMap` (Thread#threadlocals)，在调用`ThreadLocal#set`方法时，如果当前线程的 `threadlocals` 变量为空，就会创建一个 ThreadLocalMap 对象给它赋值，不为空就直接调用ThreadLocalMap 的 set 方法；
3. ThreadLocalMap 以数组的形式存储数据，数组初始大小为16，创建一个ThreadLocalMap 对象需要一个ThreadLocal对象和存储的数据对象。ThreadLocal 对象就是当前对象，根据ThreadLocal对象的hashCode计算出数组下标；
4. 当前大小大于或等于（阀值大小-阀值大小/4）时，就会扩容，扩大为原来的2倍。

再看主线程的 Looper 是如何创建的。

应用程序的入口是  ActivityThread  的方法 main：

```
    public static void main(String[] args) {
        //省略...
        // 创建主线程使用的  Looper
        Looper.prepareMainLooper();
        //省略...
        // 开始消息循环
        Looper.loop();
        //省略...
    }
```

关于 Looper 中的方法：`prepareMainLooper()`：

```
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

其实就是和上面创建当前线程的 Looper 对象一样，只不过当前线程是主线程，所以创建的 Looper  对象，只负责从主线程的 MessageQueue  中取出消息，并交给主线程的 Handler 去处理。

这里类 Looper 会把主线程使用的 Looper 对象，也就是 使用`sMainLooper`对象缓存起来。所以判断当前线程是否在主线：

```
    public static boolean isMainThread(){
        return Looper.getMainLooper().equals(Looper.myLooper());
    }
```

#### 答案

对于这个面试问题，所以回答就是：**应用程序启动的时候，会创建一个主线程使用的 Looper 对象，并把它缓存起来。Looper  对象的创建使用了类 ThreadLocal，ThreadLocal 是线程范围内的数据共享，针对某一个线程存储数据。所以就可以在当前线程中使用 `Looper.getMainLooper()`的值和`Looper.myLooper()`的值相比较，如果相等，当前线程就是主线程，否则不是主线程。**

### Looper 中的方法 loop() 是一个无限循环，为什么不会造成应用程序卡死？

关于类 Looper 中的方法 `loop()`：

```
    public static void loop() {
        final Looper me = myLooper();
       //省略...
        final MessageQueue queue = me.mQueue;
        //省略...
        for (;;) {
            Message msg = queue.next(); // might block
            //省略...
            try {
                msg.target.dispatchMessage(msg);
                //省略...
            } catch (Exception exception) {
               //省略...
            } finally {
                //省略...
            }
            //省略...
        }
    }
```

方法 loop()中所做的事情，主要就是不断的从 MessageQueue 取出消息，并交给 Handler 处理。

关于 MessageQueue 的方法 `next()`：

```
    private long mPtr; 

    @UnsupportedAppUsage
    Message next() {
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        //省略...
        int nextPollTimeoutMillis = 0;
        for (;;) {
             //省略...
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;

                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                 //省略...
                if (mQuitting) {
                    dispose();
                    return null;
                }
                 //省略...
            }
             //省略...
             // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

可以看出，方法 next() 也是一个无限循环，但是这里关键的是 natvie 方法：`nativePollOnce(ptr, nextPollTimeoutMillis);`

这个方法的作用是：

> nativePollOnce是阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。
> 当处于空闲时，往往会执行IdleHandler中的方法。当nativePollOnce()返回后，next()从mMessages中提取一个消息。

关于阻塞的含义：

> 主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

关于方法`nativePollOnce`更多分析，可以阅读**gityuan大佬**的详细分析： [Android消息机制1-Handler(Java层)](http://gityuan.com/2015/12/27/handler-message-native/#nativepollonce)。

#### 答案

对于这个面试问题，所以回答就是：**首先，类 Looper 的方法 loop()，主要是不断的从 MessageQueue 取出消息，并交给 Handler 处理。当调用 MessageQueue 的方法 next 时，会调用 native 方法 `nativePollOnce`  ，这个方法的执行，会去做判断：如果没有消息，那么CPU进入休眠状态，等到有新的消息再唤醒 CPU，也就是当 MessageQueue 的方法 enqueueMessage 往消息队列中添加一个消息时，再调用 native 方法 `nativeWake` 唤醒 CPU，继续执行。所以方法 loop() 虽然是一个无限循环，但是不会造成应用程序卡死，因为没有消息会让 CPU 进入休眠状态。**

## 总结

以上就是在面试中经常遇到的关于 Android 系统消息机制：Handler，Message，MessageQueue，Looper 相关的一些问题，只有明白了其原理，才知道回答问题的要点，也就是面试官的考察点所在。当然，这也更有助于自己对 Android 基础知识的了解。
