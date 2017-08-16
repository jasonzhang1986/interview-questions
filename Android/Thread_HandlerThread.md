### HandlerThread 原理

#### Handy class for starting a new thread that has a looper. The looper can then be used to create handler classes. Note that start() must still be called.

意思就是：HandlerThread 可以创建一个带有 looper 的线程。looper 对象可以用于创建 Handler 类来进行调度。 start 方法务必需要调用。

我们直接看 HandlerThread 的源码实现：
```Java
package android.os;


public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

    ...

}    
```
从源码可以看到，在 run 方法中创建了自己的 Looper 对象并开启了 Looper 的循环。

### 用法

```Java
HandlerThread workerThread = new HandlerThread("WorkThread");
workerThread.start();
Handler mHandler = new Handler(workerThread.getLooper());
```
如上，这样就把 Handler 和 HandlerThread中的 Looper 关联起来，可以使用 Handler 往消息队列中 post Runnable，接下来的就是 Handler 消息机制的内容了，本篇不再赘述。
