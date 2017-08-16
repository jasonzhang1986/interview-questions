## Service
A Service is not a separate process. The Service object itself does not imply it is running in its own process; unless otherwise specified, it runs in the same process as the application it is part of.
A Service is not a thread. It is not a means itself to do work off of the main thread (to avoid Application Not Responding errors).

上述是Android官网上针对Service的描述，翻译成中文大致意思如下：

Service 不是一个单独的进程，它和应用程序在同一个进程中，Service 也不是一个线程,它和线程没有任何关系，所以它不能直接处理耗时操作。如果直接把耗时操作放在 Service 的 onStartCommand() 中，很容易引起 ANR 。如果有耗时操作就必须开启一个单独的线程来处理。

## IntentService
* 默认创建工作线程，执行所有传递给 onStartCommand 的 intent，区别于 Android 的主线程
* 创建消息队列 (Handler)，将 intent 传递给 onHandleIntent 的实现，不需要担心多线程的问题
* 请求执行完成之后，自动调用 stopSelf, 不需要关心 Service 的关闭
* 提供的默认实现 onBinder 返回 null ，所以不需要重写这个方法，使用起来更简单
* 提供默认的 onStartCommand 实现，将 intent 通过 Handler 发送到消息队列，依次调用 onHandleIntent 处理

onHandleIntent 的处理内容如下：
```java
    override fun onHandleIntent(intent: Intent?) {
        Log.i(TAG, "onHandleIntent")
        if(intent==null || intent.extras==null) {
            return
        }
        val delay:Long = intent.extras.getInt("param", 2000).toLong()
        Log.i(TAG,"onHandleIntent delay = " + delay)
        try {
            Thread.sleep(delay)
        } catch (e : InterruptedException){
            e.printStackTrace()
        }

    }

```
 多次调用startService打印的日志结果如下：
 ```java
 08-15 17:49:09.104 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onCreate
08-15 17:49:09.105 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStartCommand
08-15 17:49:09.105 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStart
08-15 17:49:09.105 28840-29043/me.jifengzhang.activitytest I/IntentService: handleMessage
08-15 17:49:09.105 28840-29043/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent
08-15 17:49:09.106 28840-29043/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent delay = 2000
08-15 17:49:11.106 28840-29043/me.jifengzhang.activitytest I/IntentService: handleMessage before stopSelf
08-15 17:49:11.248 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onCreate
08-15 17:49:11.249 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStartCommand
08-15 17:49:11.249 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStart
08-15 17:49:11.249 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage
08-15 17:49:11.249 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent
08-15 17:49:11.249 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent delay = 4000
08-15 17:49:12.618 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStartCommand
08-15 17:49:12.618 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStart
08-15 17:49:15.152 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStartCommand
08-15 17:49:15.152 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStart
08-15 17:49:15.250 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage before stopSelf
08-15 17:49:15.251 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage
08-15 17:49:15.251 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent
08-15 17:49:15.252 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent delay = 6000
08-15 17:49:18.264 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStartCommand
08-15 17:49:18.264 28840-28840/me.jifengzhang.activitytest I/MyIntentService: onStart
08-15 17:49:21.252 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage before stopSelf
08-15 17:49:21.254 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage
08-15 17:49:21.254 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent
08-15 17:49:21.255 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent delay = 8000
08-15 17:49:29.256 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage before stopSelf
08-15 17:49:29.258 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage
08-15 17:49:29.258 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent
08-15 17:49:29.258 28840-29054/me.jifengzhang.activitytest I/MyIntentService: onHandleIntent delay = 10000
08-15 17:49:39.260 28840-29054/me.jifengzhang.activitytest I/IntentService: handleMessage before stopSelf
 ```

第一次 startService 后等待超过2秒之后再次调用 startService, 我们看到重新走了 Service 的生命周期，之后多次快速调用 startService，看到一次发送到消息队列，然后依次单线程执行 onHandelIntent, 最后在 stopSelf。


这里有个细节，每次 handleMessage 中在 onHandleIntent 之后都调用了 stopSelf ，为什么还可以执行后续的 intent 操作，跟踪源码
```Java
//Service.java
public final void stopSelf(int startId) {
    if (mActivityManager == null) {
        return;
    }
    try {
        mActivityManager.stopServiceToken(
                new ComponentName(this, mClassName), mToken, startId);
    } catch (RemoteException ex) {
    }
}

//ActivityManagerService.java
@Override
public boolean stopServiceToken(ComponentName className, IBinder token,
        int startId) {
    synchronized(this) {
        //mServices 的声明： final ActiveServices mServices;
        return mServices.stopServiceTokenLocked(className, token, startId);
    }
}


boolean stopServiceTokenLocked(ComponentName className, IBinder token,
        int startId) {
    if (DEBUG_SERVICE) Slog.v(TAG, "stopServiceToken: " + className
            + " " + token + " startId=" + startId);
    ServiceRecord r = findServiceLocked(className, token, UserHandle.getCallingUserId());
    if (r != null) {
        if (startId >= 0) {

            ...

            //重点看这里，判断最后一个 lastStartId 不等于 stopSelf 的参数时，直接 return false
            if (r.getLastStartId() != startId) {
                return false;
            }

            if (r.deliveredStarts.size() > 0) {
                Slog.w(TAG, "stopServiceToken startId " + startId
                        + " is last, but have " + r.deliveredStarts.size()
                        + " remaining args");
            }
        }

        synchronized (r.stats.getBatteryStats()) {
            r.stats.stopRunningLocked();
        }

        ...

        return true;
    }
    return false;
}               

```
