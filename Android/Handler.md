##### Android平台上，主要用到的通信机制有两种：Handler和Binder，前者用于进程内部的通信，后者主要用于跨进程通信。

#### 1. 概述

今天我们主要来聊一聊进程内部的消息机制**Handler**。

从技术实现来说消息机制并不复杂，不只是Android平台,各种平台的消息机制原理基本上都是比较相似的，其中用到的主要概念有：
1. 消息发送者
2. 消息队列
3. 消息循环处理

简单示意图如下：

![image](http://7u2jwu.com1.z0.glb.clouddn.com/android/handler/xiaoxi01.png)

图中表达的意思是，消息发送者通过某种方式将消息发送到消息队列中，同时还有一个消息处理循环，不断从消息队列里取消息，并进行处理。

Android的消息机制主要是指Handler的运行机制，Hander的运行需要相关的MessageQueue和Looper的支撑。图中右侧的部分可以理解为Android中的Looper类，这个类的内部有对应的消息队列(MessageQueue mQueue)和loop方法(取消息的循环)

从我们开发者的角度来说，Handler是Android消息机制的最上层接口，这使得我们在开发过程中绝大多数情况下只需要和Handler打交道就可以，Handler的使用方法也很简单，我们常常在子线程中需要更新UI时使用，常见代码如下：

```java
private Handler handler = new Handler(){
    @Override public void handleMessage(Message msg) {
      super.handleMessage(msg);
      switch (msg.what) {
        case xxx:
          ...
          updateUI();
          break;
      }
    }
  };

  ....

  private void threadMethod() {
    new Thread(){
      @Override public void run() {
        super.run();
        ...
        handler.sendMessage(msg);
        ...
      }
    }.start();
  }

```
在子线程中创建并使用Handler：
```java
 //在子线程中使用Handler
   class LooperThread extends Thread {
        public Handler mHandler;

        public void run() {
            Looper.prepare();

            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };

            Looper.loop();
        }
    }
```

从上面两个例子，我们可以看到通过handler的sendMessage方法来发送消息，通过Looper.loop方法来进行消息循环；至于消息处理是隐藏在Loop和MessageQueue中，我们接下来会一点点来深入分析。



说到Handler我们先来看下它的构造方法，开发过程中经常用到的构造方法是Handler()和Handler(Looper looper);

```java
public class Handler{
  	final MessageQueue mQueue;
    final Looper mLooper;
    final Callback mCallback;
  	final boolean mAsynchronous;

  	.....

	public Handler() {
        this(null, false);
    }

    public Handler(Looper looper) {
        this(looper, null, false);
    }

    public Handler(Callback callback, boolean async) {
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
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

   public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
  ......

 }
```
通过如上代码我们可以看到构造方法目的都是为了给mLooper、mQueue、mCallback和mAsyncronous赋值，callback和async这两个我们用的较少，我们此次重点关注mLooper和mQueue。

```java
Handler handler = new Handler(Looper.getMainLooper());
```

如上是我们常用的指定了Looper的构造方法，内部实现很简单，直接就赋值：

```java
	/**
     * Use the provided {@link Looper} instead of the default one.
     *
     * @param looper The looper, must not be null.
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }

	public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

另一个更常用的构造方法是如下这种：

```java
Handler handler = new Handler();

public class Handler {
   ....

    /**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }

   public Handler(Callback callback, boolean async) {
        .....
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
  ....
}

```

因为默认构造方法没有入参，那么mLooper通过Looper自身的myLooper方法来获取，判断mLooper是否为null，如果非空则给mQueue、callback等赋值，如果等于null则说明Handler是在线程中创建并且创建前没有调用Looper的prepare()方法；那我们猜想肯定是在Looper.prepare()方法中初始化了Looper对象。

这里有两个疑问

* 在Activity(UI线程)中new Handler()的时候也没有去调用Looper.prepare()，为什么没有抛异常？
* Looper.prepare()是如何初始化Looper对象并在通过myLooper方法向外部提供引用的？

我们先来看第二个问题，还是老方法直接看源码实现：

```java
public class Looper {
  	// sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
 	final MessageQueue mQueue;
    final Thread mThread;
    volatile boolean mRun;

  	.....

	 /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

  	....

   private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mRun = true;
        mThread = Thread.currentThread();
    }

   /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
}
```

代码实现很简单，就是new一个Looper对象set给sThreadLocal，然后获取是通过sThreadLocal的get方法。这样我们明白了为什么在thread中new Handler前要调用Looper.parepare()。

接下来我们回头看问题1，为什么在UI线程中可以直接new Handler()来使用呢？UI线程直接初始化Handler没有抛异常说明sThreadLocal.get()返回值不是空，也就是已经set过Looper对象，那是哪里设置的呢？

了解应用启动流程的应该知道，应用的main方法在ActivityThread中，在main方法中完成了一些初始化工作，我们来看main方法中做了什么
```java
  public static void main(String[] args) {

    ...

    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if(sMainThreadHandler == null) {
      sMainThreadHandler = thread.getHandler();
    }

    ...

    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
  }
```
这里我们看到了熟悉的Looper.prepare和Looper.loop，不过这里调用的是Looper的另一个方法prepareMainLooper，跟进去看

```java
  public static void prepareMainLooper() {
    prepare(false);
    Class var0 = Looper.class;
    synchronized(Looper.class) {
      if(sMainLooper != null) {
        throw new IllegalStateException("The main Looper has already been prepared.");
      } else {
        sMainLooper = myLooper();
      }
    }
  }

  public static void prepare() {
    prepare(true);
  }

  private static void prepare(boolean quitAllowed) {
    if(sThreadLocal.get() != null) {
      throw new RuntimeException("Only one Looper may be created per thread");
    } else {
      sThreadLocal.set(new Looper(quitAllowed));
    }
  }
```
可以看到prepareMainLooper()和prepare()调用的是Looper的同一个方法prepard(boolean quitAllowed)，UI线程的Looper是不允许退出的(如果允许退出，调用了quit岂不UI就无法刷新啦)，其他线程的Looper是允许退出的。

所以，UI线程并非特殊，而是在更早的时候系统已经做好了Looper的初始化操作。既然已经发现了系统初始化Looper的地方，那UI线程是不是也有自己的Handler来处理消息呢？答案显示是有的。

prepare方法调用后，创建Handler,这里是new ActivityThread对象，ActivityThread有实例变量mH就是Handler的对象
```java
main() {
    ...
    ActivityThread thread = new ActivityThread();

    if(sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    ...
}

final ActivityThread.H mH = new ActivityThread.H(null);

final Handler getHandler() {
    return this.mH;
}

...

private class H extends Handler {

    ...
}
```
#### 2. 消息循环

到这里**Looper**和**Handler**都准备好，还有个必须要做的就是开启消息循环**Looper.loop()**。

```java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

		...

        for (;;) { //永久循环
            Message msg = queue.next(); // might block
            if (msg == null) { //死循环退出条件
                // No message indicates that the message queue is quitting.
                return;
            }

            ...

            msg.target.dispatchMessage(msg);//这里的target是Handler

            ...

        }
    }
```

说到消息的循环，就不得不提MessageQueue中next方法

```java
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
           ......

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                   // 如果从队列里拿到的msg是个“同步分割栏”，那么就寻找其后第一个“异步消息”
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
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
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

              ......

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

调用这个方法的时候，有可能让线程进入等待状态。什么情况下线程会进入等待状态呢？有如下两种情况

1. 当消息队列中没有消息是，它会使线程进入等待状态
2. 消息队列中有消息，但是消息指定了执行的时间，而现在还没有到这个时间，线程也会进入等待状态

消息队列中的消息是按照时间先后来排序的，后面我们在分析消息发送的时候会看到。

大家可能注意到如下一条native的语句，它是查看当前消息队列中有没有消息：

```java
nativePollOnce(mPtr, nextPollTimeoutMillis);
```

这是一个JNI方法，我们等一下再分析，这里传入的参数mPtr就是指向前面我们在JNI层创建的NativeMessageQueue对象了，而参数nextPollTimeoutMillis则表示如果当前消息队列中没有消息，它要等待的时候，for循环开始时，传入的值为0，表示不等待。

当前nativePoolOnce返回后，就去看看消息队列中有没有消息：

```java
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
        if (false) Log.v("MessageQueue", "Returning message: " + msg);
        return msg;
    }
} else {
    // No more messages.
    nextPollTimeoutMillis = -1;
}
```

如果消息队列中有消息，并且当前时候大于等于消息中的执行时间，那么就直接返回这个消息给Looper.loop消息处理，否则的话就要等待到消息的执行时间：

```java
nextPollTimeoutMillis = (int) Math.min(when - now, Integer.MAX_VALUE);
```

如果消息队列中没有消息，那就进入到无穷等待状态直到有新消息：

```java
nextPollTimeoutMillis = -1;  
```

-1表示下次调用nativePollOnce时，如果消息中没有消息就进入无限等待状态中去。这里计算出来的等待时间是在下次调用nativePollOnce的时候使用。

 这里说的等待，是空闲等待，而不是忙等待，因此，在进入空闲等待状态前，如果应用程序注册了IdleHandler接口来处理一些事情，那么就会先执行这里IdleHandler，然后再进入等待状态。

```java
 // If first time idle, then get the number of idlers to run.
// Idle handles only run if the queue is empty or if the first message
// in the queue (possibly a barrier) is due to be handled in the future.
if (pendingIdleHandlerCount < 0  && (mMessages == null || now < mMessages.when)) {
  	pendingIdleHandlerCount = mIdleHandlers.size();
}
if (pendingIdleHandlerCount <= 0) {
  	// No idle handlers to run.  Loop and wait some more.
  	mBlocked = true;
 	continue;
}

if (mPendingIdleHandlers == null) {
  	mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
}
mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
```

如果没有IdleHandler即pendingIdleHandlerCount等于0，那么下面的逻辑就不执行了，通过continue语句直接进入下一次循环，否则就把注册的mPendingIdleHandlers中的IdleHandler取出来，放到mPendingIdleHandler数组中，并循环执行这些注册了的IdelHandler：

```java
// Run the idle handlers.
// We only ever reach this code block during the first iteration.
for (int i = 0; i < pendingIdleHandlerCount; i++) {
    final IdleHandler idler = mPendingIdleHandlers[i];
    mPendingIdleHandlers[i] = null; // release the reference to the handler

    boolean keep = false;
    try {
   		 keep = idler.queueIdle();
    } catch (Throwable t) {
    	 Log.wtf("MessageQueue", "IdleHandler threw exception", t);
    }

    if (!keep) {
    	synchronized (this) {
    		mIdleHandlers.remove(idler);
    	}
    }
}
```

执行完这些IdleHandler之后，下次再次调用nativePollOnce函数的时候，就不设置超时时间了，因为很有可能在执行IdleHandler的时候，已经有新的消息加入到消息队列中去了，因此，要重置nextPollTimeoutMillis的值：

```java
// While calling an idle handler, a new message could have been delivered
// so go back and look again for a pending message without waiting.
nextPollTimeoutMillis = 0;
```

分析完MessageQueue的next方法之后，我们来稍微深入的分析下JNI方法nativePollOnce了，看看它是如何进入等待状态的，这个函数在[frameworks/base/core/jni/android_os_MessageQueue.cpp](http://androidxref.com/4.2_r1/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)文件中：

```c
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jint ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, timeoutMillis);
}

static JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()V", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "()V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(II)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(I)V", (void*)android_os_MessageQueue_nativeWake }
};
```

这个函数首先是通过传进来的参数ptr取回在Java层创建的MessageQueue对象时在JNI层创建的NativeMessageQueue对象：

```java
 public final class MessageQueue {
 	private long mPtr; // used by native code
   ....
   	MessageQueue(boolean quitAllowed) {
       mQuitAllowed = quitAllowed;
       mPtr = nativeInit();
    }
   .....
 }
```

```c
static void android_os_MessageQueue_nativeInit(JNIEnv* env, jobject obj) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return;
    }

    nativeMessageQueue->incStrong(env);
    android_os_MessageQueue_setNativeMessageQueue(env, obj, nativeMessageQueue);
}
```

取到NativeMessageQueue对象，然后调用它的pollOnce函数

```c
void NativeMessageQueue::pollOnce(JNIEnv* env, int timeoutMillis) {
    mInCallback = true;
    mLooper->pollOnce(timeoutMillis);
    mInCallback = false;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

这里把操作转给mLooper对象的pollOnce函数处理，这里的mLooer对象是C++层的对象，它也是在JNI层创建NativeMessageQueue对象时创建的：

```c
NativeMessageQueue::NativeMessageQueue() : mInCallback(false), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

它的pollOnce函数在Looper.cpp([JellyBean](http://androidxref.com/4.2_r1/xref/frameworks/native/libs/utils/Looper.cpp)、[Lollipop](http://androidxref.com/5.0.0_r2/xref/system/core/libutils/Looper.cpp))中:

```c
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        ....
        if (result != 0) {
			.....
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```

我们只看核心部分，这里主要是继续调用pollInner函数来进一步操作，如果pollInner返回不等于0，这个函数就可以返回了。

pollInner的定义如下：

```c
int Looper::pollInner(int timeoutMillis) {
	.....

    // Poll.
    int result = ALOOPER_POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // Acquire lock.
    mLock.lock();

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = ALOOPER_POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
		.....
        result = ALOOPER_POLL_TIMEOUT;
        goto Done;
    }

	.....

    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
           .....
        }
    }
Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

				....
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = ALOOPER_POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
   ......

    return result;
}
```

这个函数比较长并且其中用到了goto，我们仍然只关注核心代码，我们看到函数开头首先是调用epoll_wait函数来看看epoll专用文件描述符mEpollFd所监控的文件描述符是否有IO事件发生，它设置监控的超时时间为pollOnce函数传递过来的timeoutMillis：

```c
int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
```

当mEpollFd所监控的文件描述符发生了要监控的IO事件后或者监控时间超时后，线程就从epoll_wait返回了，否则线程就会在epoll_wait函数中进入睡眠状态了。返回后如果eventCount等于0，就说明是超时了：

```c
if (eventCount == 0) {  
    ......  
    result = ALOOPER_POLL_TIMEOUT;  
    goto Done;  
}  
```

如果eventCount不等于0，就说明发生要监控的事件：

```c
for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
           .....
        }
    }
```

这里我们只关注mWakeReadPipeFd文件描述符上的事件, 在Looper的构造函数中设置了要监控mWakeReadPipeFd文件描述符的EPOLLIN事件：

```c
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    int result = pipe(wakeFds); // 创建一个管道
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

    mWakeReadPipeFd = wakeFds[0];   // 管道的“读取端”
    mWakeWritePipeFd = wakeFds[1];  // 管道的“写入端”

    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);

    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);

    // Allocate the epoll instance and register the wake pipe.
    // 创建一个epoll
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union

    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    // 监听管道的read端
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",errno);
}
```

如果在mWakeReadPipeFd文件描述符上发生了EPOLLIN就说明应用程序中的消息队列里面有新的消息需要处理了，接下来它就会先调用awoken函数清空管道中的内容，以便下次再调用pollInner函数时，知道自从上次处理完消息队列中的消息后，有没有新的消息加进来。

awoken函数的实现很简单，它就是把管道中的内容都读取出来：

```c
void Looper::awoken() {
	.....
    char buffer[16];
    ssize_t nRead;
    do {
        nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
    } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
}
```

因为当其它的线程向应用程序的消息队列加入新的消息时，会向这个管道写入新的内容来通知应用程序主线程有新的消息需要处理了，下面我们分析消息的发送的时候将会看到。

 这样，消息的循环过程就分析完了，这部分逻辑还是比较复杂的，它利用Linux系统中的管道（pipe）进程间通信机制来实现消息的等待和处理，不过，了解了这部分内容之后，下面我们分析消息的发送和处理就简单多了。



#### 3. 消息发送

在线程(不管是UI线程还是子线程)中准备好消息队列并且进入消息循环后，其他地方就可以往这个消息队列中发送消息了。

在前面我们提到过Handler的构造方法，主要就是初始化类成员mLooper和mQueue。

```java
public class Handler {  
    ......  

    public Handler() {  
        ......  

        mLooper = Looper.myLooper();  
        ......  

        mQueue = mLooper.mQueue;  
        ......  
    }  


    final MessageQueue mQueue;  
    final Looper mLooper;  
    ......  
}  
```

有了Looper对象后，就可以通过mLooper.mQueue来访问消息队列了。发送消息的方法大家都不陌生，就是常用的sendMessage方法。在发送消息时，是可以指定消息的处理时间的，但是通过sendMessage函数发送的消息的处理时间默认就为当前时间，即表示要马上处理，因此，从sendMessage函数中调用sendMessageDelayed函数，传入的时间参数为0，表示这个消息不要延时处理，而在sendMessageDelayed函数中，则会先获得当前时间，然后加上消息要延时处理的时间，即得到这个处理这个消息的绝对时间，然后调用sendMessageAtTime函数来把消息加入到应用程序的消息队列中去。

 在sendMessageAtTime函数，首先得到应用程序的消息队列mQueue，这是在Handler对象构造时初始化好的，前面已经分析过了，接着设置这个消息的目标对象target，即这个消息最终是由谁来处理的：

```java
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

这里将它赋值为this，即表示这个消息最终由这个Handler对象来处理。

函数最后调用queue.enqueueMessage来把这个消息加入到消息队列中，具体实现在[MessageQueue.java](http://androidxref.com/4.2_r1/xref/frameworks/base/core/java/android/os/MessageQueue.java)文件中：

```java
    final boolean enqueueMessage(Message msg, long when) {
        ......

        boolean needWake;
        synchronized (this) {
           .......

            msg.when = when;
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
              // 此时，新消息会插入到链表的表头，这意味着队列需要调整唤醒时间啦。
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
              // 此时，新消息会插入到链表的内部，一般情况下，这不需要调整唤醒时间。
              // 但还必须考虑到当表头为“同步分隔栏”的情况
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                      // 说明即便msg是异步的，也不是链表中第一个异步消息，所以没必要唤醒了
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
        }
        if (needWake) {
            nativeWake(mPtr);
        }
        return true;
    }
```

从这段代码来看MessageQueue说是一个消息队列，但它其实不是队列，而是一个链表。

把消息加入到消息队列时，分两种情况，一种当前消息队列为空时，这时候线程一般就是处于空闲等待状态了，这时候就要唤醒它，另一种情况是消息队列不为空，这时候就不需要唤醒线程了，因为这时候它一定是在忙着处于消息队列中的消息，因此不会处于空闲等待的状态。

第一种情况比较简单，只要把消息放在消息队列头就可以了：

```java
msg.next = p;  
mMessages = msg;  
needWake = mBlocked;
```

 第二种情况相对就比较复杂一些了，前面我们说过，当往消息队列中发送消息时，是可以指定消息的处理时间的，而消息队列中的消息，就是按照这个时间从小到大来排序的，因此，当把新的消息加入到消息队列时，就要根据它的处理时间来找到合适的位置，然后再放进消息队列中去：

```java
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
```

把消息加入到消息队列去后，如果线程正处于空闲等待状态，就需要调用natvieWake函数来唤醒它了，这是一个JNI方法，定义在[android_os_MessageQueue.cpp](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)文件中：

```c
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->wake();
}
```

这个JNI层的NativeMessageQueue对象我们在前面分析消息循环的时候创建好的，保存在Java层的MessageQueue对象的mPtr成员变量中，这里把它取回来之后，就调用它的wake函数来唤醒关联的线程，wake方法也在[android_os_MessageQueue.cpp](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/jni/android_os_MessageQueue.cpp)文件中：

```c
void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

 这里它又通过成员变量mLooper的wake函数来执行操作，这里的mLooper成员变量是一个C++层实现的Looper对象，它定义在[Looper.cpp](http://androidxref.com/5.0.0_r2/xref/system/core/libutils/Looper.cpp)文件中：

```c
void Looper::wake() {
	......
    ssize_t nWrite;
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);

    ......
}
```

 这个wake函数很简单，只是通过打开文件描述符mWakeWritePipeFd往管道的写入一个"W"字符串。其实，往管道写入什么内容并不重要，往管道写入内容的目的是为了唤醒关联的线程。前面我们在分析应用程序的消息循环时说到，当消息队列中没有消息处理时，关联的线程就会进入空闲等待状态，而这个空闲等待状态就是通过调用这个Looper类的pollInner函数来进入的，具体就是在pollInner函数中调用epoll_wait函数来等待管道中有内容可读的。

这时候既然管道中有内容可读了，关联的线程就会从这里的Looper类的pollInner函数返回到JNI层的nativePollOnce函数，最后返回到Java层中的MessageQueue.next方法中去，这里它就会发现消息队列中有新的消息需要处理了，于就会处理这个消息。

#### 4. 消息处理

前面第一部分分析消息循环时，在Looper类的looper方法中进行消息循环的，这个方法中从消息队列中获取到消息对象msg后，就会调用它的target成员变量的dispatchMessage方法来处理这个消息。
```java
    public static void loop() {
        final Looper me = myLooper();
      	......
        final MessageQueue queue = me.mQueue;
		......
        for (;;) {
            Message msg = queue.next(); // might block

          	.......

            msg.target.dispatchMessage(msg);

            ......

            msg.recycleUnchecked();
        }
    }
```
在前面分析消息的发送时说过，这个消息对象msg的成员变量target是在发送消息的时候设置好的，通过哪个Handler来发送消息，就通过哪个Handler来处理消息。
```java
public class Handler {
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
}
```

这里的消息对象msg的callback成员变量和Handler类的mCallBack成员变量一般都为null，于是，就会调用Handler类的handleMessage方法来处理这个消息，这就到了我们平常使用Handler的时候重写的Handler的方法handlerMessage。



#### 5. 总结

 至此，我们就从消息循环、消息发送和消息处理三个部分分析完Android应用程序的消息处理机制了，为了更深理解，这里我们对其中的一些要点作一个总结：

* Android应用程序的消息处理机制由消息循环、消息发送和消息处理三个部分组成的。
* Android应用程序的主线程在进入消息循环过程前，会在内部创建一个Linux管道（Pipe），这个管道的作用是使得Android应用程序主线程在消息队列为空时可以进入空闲等待状态，并且使得当应用程序的消息队列有消息需要处理时唤醒应用程序的主线程。
* Android应用程序的主线程进入空闲等待状态的方式实际上就是在管道的读端等待管道中有新的内容可读，具体来说就是是通过Linux系统的Epoll机制中的epoll_wait函数进行的。
* 当往Android应用程序的消息队列中加入新的消息时，会同时往管道中的写端写入内容，通过这种方式就可以唤醒正在等待消息到来的应用程序主线程。
* 当应用程序主线程在进入空闲等待前，会认为当前线程处理空闲状态，于是就会调用那些已经注册了的IdleHandler接口，使得应用程序有机会在空闲的时候处理一些事情。

​    

#### 6. 问题


最后，在前面的Looper部分相信很多人有个疑问

- sThreadLocal在Looper类中是静态的，为什么线程A中已经调用了Looper.prepare给sThreadLocal设置了值，在线程B中直接调用Looper.myLooper从sThreadLocal中get值返回确是null？
- 一个线程可以有几个Handler？
- 一个线程可以有几个Looper？

#### 7. 补充内容

##### 7.1 ThreadLocal

官方文档中对ThreadLocal的说明：

```java
/**
 * Implements a thread-local storage, that is, a variable for which each thread
 * has its own value. All threads share the same {@code ThreadLocal} object,
 * but each sees a different value when accessing it, and changes made by one
 * thread do not affect the other threads. The implementation supports
 * {@code null} values.
 *
 * @see java.lang.Thread
 */
public class ThreadLocal<T> {
}
```

这段话的意思是实现了一个线程相关的存储，即每个线程都有自己独立的变量。所有的线程都共享者这一个`ThreadLocal`对象，并且当一个线程的值发生改变之后，不会影响其他的线程的值。

ThreadLocal的类定义使用了泛型`ThreadLocal<T>`，其中T指代的是在线程中存取值的类型。（对应Android中使用的ThreadLocal, T则存放的类型为Looper）

* set方法

```java
    /**
     * Sets the value of this variable for the current thread. If set to
     * {@code null}, the value will be set to null and the underlying entry will
     * still be present.
     *
     * @param value the new value of the variable for the caller thread.
     */
    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }

    /**
     * Sets entry for given ThreadLocal to given value, creating an
     * entry if necessary.
     */
    void put(ThreadLocal<?> key, Object value) {
      cleanUp();

      // Keep track of first tombstone. That's where we want to go back
      // and add an entry if necessary.
      int firstTombstone = -1;

      for (int index = key.hash & mask;; index = next(index)) {
        Object k = table[index];

        if (k == key.reference) {
          // Replace existing entry.
          table[index + 1] = value;
          return;
        }

        if (k == null) {
          if (firstTombstone == -1) {
            // Fill in null slot.
            table[index] = key.reference;
            table[index + 1] = value;
            size++;
            return;
          }

          // Go back and replace first tombstone.
          table[firstTombstone] = key.reference;
          table[firstTombstone + 1] = value;
          tombstones--;
          size++;
          return;
        }

        // Remember first tombstone.
        if (firstTombstone == -1 && k == TOMBSTONE) {
          firstTombstone = index;
        }
      }
    }
```

简单地说，每个线程对象内部会记录一张逻辑上的key-value表，当我们在不同Thread里调用Looper.prepare()时，其实是向Thread对应的那张表里添加一个key-value项，其中的key部分，指向的是同一个对象，即Looper.sThreadLocal静态对象，而value部分，则彼此不同，我们可以画出如下示意图：

![ThreadLocal示意图](http://7u2jwu.com1.z0.glb.clouddn.com/android/handler/ThreadLocal.png)

看到了吧，不同Thread会对应不同Object[]数组，该数组以每2个元素为一个key-value对。请注意不同Thread虽然使用同一个静态对象作为key值，最终却会对应不同的Looper对象。

##### 7.2 同步分隔栏

上面的代码中还有一个“同步分割栏”的概念需要提一下。所谓“同步分割栏”，可以被理解为一个特殊Message，它的target域为null。它不能通过sendMessageAtTime()等函数加入消息队列里，而只能通过调用Looper的postSyncBarrier()来加入消息队列。

“同步分割栏”是起什么作用的呢？它就像一个卡子，卡在消息链表中的某个位置，当消息循环不断从消息链表中摘取消息并进行处理时，一旦遇到这种“同步分割栏”，那么即使在分割栏之后还有若干已经到时的普通Message，也不会摘取这些消息了。请注意，此时只是不会摘取“普通Message”了，如果队列中还设置有“异步Message”，那么还是会摘取已到时的“异步Message”的。

在Android的消息机制里，“普通Message”和“异步Message”也就是这点儿区别啦，也就是说，如果消息列表中根本没有设置“同步分割栏”的话，那么“普通Message”和“异步Message”的处理就没什么大的不同了。

打入“同步分割栏”的postSyncBarrier()函数的代码如下：
[frameworks/base/core/java/android/os/Looper.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/Looper.java)

```java
public int postSyncBarrier() {
    return mQueue.enqueueSyncBarrier(SystemClock.uptimeMillis());
}
```

[frameworks/base/core/java/android/os/MessageQueue.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/MessageQueue.java)

```java
int enqueueSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
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
        if (prev != null) {
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

要得到“异步Message”，只需调用一下Message的setAsynchronous()即可：
[frameworks/base/core/java/android/os/Message.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/Message.java)

```java
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

一般，我们是通过“异步Handler”向消息队列打入“异步Message”的。异步Handler的mAsynchronous域为true，因此它在调用enqueueMessage()时，可以走入：

```java
if (mAsynchronous) {
    msg.setAsynchronous(true);
}
```

现在我们画一张关于“同步分割栏”的示意图：

![img](http://7u2jwu.com1.z0.glb.clouddn.com/android/handler/handler-async.png)

图中的消息队列中有一个“同步分割栏”，因此它后面的“2”号Message即使到时了，也不会摘取下来。而“3”号Message因为是个异步Message，所以当它到时后，是可以进行处理的。

“同步分割栏”这种卡子会一直卡在消息队列中，除非我们调用removeSyncBarrier()删除这个卡子。
[frameworks/base/core/java/android/os/Looper.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/Looper.java)

```java
public void removeSyncBarrier(int token) {
    mQueue.removeSyncBarrier(token);
}
```

[frameworks/base/core/java/android/os/MessageQueue.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/MessageQueue.java)

```java
void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycle();

        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

和插入消息类似，如果删除动作改变了链表的头部，也意味着队列的最近唤醒时间应该被调整了，因此needWake会被设为true，以便代码下方可以走进nativeWake()。



#### 参考文献

[Android应用程序消息处理机制（Looper、Handler）分析](http://blog.csdn.net/luoshengyang/article/details/6817933)

[聊一聊Android的消息机制](https://my.oschina.net/youranhongcha/blog/492591)

[Android消息机制2-Handler(Native层)](http://gityuan.com/2015/12/27/handler-message-native/)

#### 相关源码

[Handler.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/Handler.java)

[Looper.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/Looper.java)

[MessageQueue.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/MessageQueue.java)

[Message.java](http://androidxref.com/5.0.0_r2/xref/frameworks/base/core/java/android/os/Message.java)

[Looper.cpp](http://androidxref.com/5.0.0_r2/xref/system/core/libutils/Looper.cpp)
