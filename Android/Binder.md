## 概述
Android四大组件分别是 **Activity**、**Service**、**BroadcastReceiver**、**ContentProvider**，我们平时开发的 App 都是由四大组件中的一个或者多个组合而成；这四大组件所涉及的多进程间通信底层实现都是基于 Binder 的 IPC 机制。

我们平时开发过程中用到很多跨进程的通信，比如：
  1. App 中的 ActivityA 调用系统的 ActivityB

  2. App 中的 ActivityA 调用另一个 App 的 Service

Binder 作为 Android 系统提供的一种的 IPC 机制，整个 Android 系统架构中大量采用了 Binder 机制的 IPC 方案。
## IPC
说到 IPC，我们来稍微看一下 IPC 机制的原理：**进程间通信（IPC, Inter-Process Communication）** 指至少两个进程或线程减传送数据或者信号的技术或方法。每个进程都有自己的一部分独立的系统资源，彼此之间是隔离的。为了能使不同的进程相互访问资源并进行协调工作，才有了进程间通信。IPC 的方法有多种，包括**文件、Socket、管道、信号量、共享内存** 等，我们今天主要来说说 Android 中的 Binde r。

## Binder
### 概念
每个 Android 在各自独立的 Dalvik 虚拟机中运行，每个进程拥有独立的地址空间和资源，对应的地址空间有一部分是用户空间，另一部分是内核空间。对于用户空间，不同进程之间彼此是不能共享的，而内核空间则是可以共享的，Client 进程同 Server 进程通信，就是利用进程间可共享的内核空间(*内核驱动*)来完成底层通信工作的。

![IPC 机制](http://7u2jwu.com1.z0.glb.clouddn.com/blog/20170502/134208172.png)

Android 中 Binder 通信采用的是 C/S 架构，Binder 框架包括 Client、Server、ServiceManager 以及 Binder 驱动，其中 ServiceManager 用来管理系统中各种服务。其中 Server，Client，ServiceManager 运行于用户空间，Binder 驱动运行于内核空间。这四个角色的关系和互联网类似：Server 是服务器，Client 是客户终端，ServiceManager 是域名服务器（DNS），Binder 驱动是路由器。如图：

![Binder](http://7u2jwu.com1.z0.glb.clouddn.com/blog/20170502/134549061.png)

### Binder使用实例
Binder 是 Android 中一个类，它实现 IBinder 接口，从 IPC 方面来看 Binder 是一种跨进程通信的方式，从 Framework 方面来看 Binder 是 ServiceManager 连接各种 Manager (WindowManager, ActivityManager 等等)和相应的 ManagerService (WindowManagerService, ActivityManagerService 等等)的桥梁；
![AMS 架构](http://7u2jwu.com1.z0.glb.clouddn.com/blog/20170504/093603769.png)
对客户端应用来说，Binder 是客户端和服务端通信的媒介，当bindService的时候，服务端会返回一个包含服务端业务调用的 Binder 对象，通过这个 Binder 对象客户端可以获取服务端提供的服务、数据等。
```java
public class LocalService extends Service {
    // Binder given to clients
    private final IBinder mBinder = new LocalBinder();
    // Random number generator
    private final Random mGenerator = new Random();

    /**
     * Class used for the client Binder.  Because we know this service always
     * runs in the same process as its clients, we don't need to deal with IPC.
     */
    public class LocalBinder extends Binder {
        LocalService getService() {
            // Return this instance of LocalService so clients can call public methods
            return LocalService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder; //通过 onBind 返回一个服务端的 Binder 对象
    }

    /** method for clients */
    public int getRandomNumber() {
      return mGenerator.nextInt(100);
    }
}
```
上面这段代码是 Service 的一种实现方式，这里的 Binder 不涉及跨进程通信，所以比较简单，跨进程的 Binder 使用在 Android 中主要是 Messager 和 AIDL，Messager 的底层实现也是 AIDL，所以我们先来聊聊 AIDL。

### AIDL

新建一个 AIDL 的文件</br>
![新建 AIDL File](http://7u2jwu.com1.z0.glb.clouddn.com/blog/20170503/153750191.png)

新建后会自动生成 aidl 的目录，并将新建的 AIDL 文件放到该目录下</br>
![生成的aidl目录](http://7u2jwu.com1.z0.glb.clouddn.com/blog/20170503/161218200.png)

Book.aidl 文件内容是自动生成的，我们可以根据自己的需求进行修改, 自动生成的代码标记了 import 语句的地方和可以用做 AIDL 参数和返回值的一些基本类型
```java
// Book.aidl
package me.jifengzhang.aidldemo;

// Declare any non-default types here with import statements

interface Book {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

这个demo主要是提供 Book 的两个操作，我们新建的文件包括 Book.java 、 IBook.aidl 、 IBookManager.aidl，其中 Book 对象需要跨进程通过Binder传输，所以需要实现 Parcelable 接口。

**Book.java**
```java
import android.os.Parcelable;

public class Book implements Parcelable{
    public int bookId;
    public String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    protected Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
}
```
**Book.aidl**
```java
package me.jifengzhang.aidldemo;

parcelable Book;
```
**IBookManager.aidl**
```java
package me.jifengzhang.aidldemo;

import me.jifengzhang.aidldemo.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```
编译后，会自动生成 IBookManager.aidl 对应的 java 文件(app/build/generated/source/aidl/debug/me/jifengzhang/aidldemo/IBookManager.java)，格式调整后如下：
```java
package me.jifengzhang.aidldemo;

public interface IBookManager extends android.os.IInterface {
    /**
    * Local-side IPC implementation stub class.
    */
   public static abstract class Stub extends android.os.Binder implements me.jifengzhang.aidldemo.IBookManager {

       ...

   }

   public java.util.List<me.jifengzhang.aidldemo.Book> getBookList() throws android.os.RemoteException;

   public void addBook(me.jifengzhang.aidldemo.Book book) throws android.os.RemoteException;
```

这个类结构其实很简单，它继承了 IInterface 接口（所有可以在 Binder 中传输的接口都需要继承 IInterface 接口）同时它自己也是一个接口类，可以看到其实它的内部结构就是声明了两个方法 getBookList 和 addBook (显然这两个方法是 IBookManager.aidl 中声明的)，同时还有一个内部类 Stub，这个类就是一个 Binder 类（很明显 Stub 继承 Binder）。但我们应该认识到 IBookManager.java 的核心实现是 Stub 以及 Stub 的内部代理类 Proxy。

下面详细介绍下 Stub 以及内部代理类 Proxy 中各个方法的含义：

** DESCRIPTOR **

Binder 的唯一标识符，一般使用当前 Binder 的类名表示， 比如上述例子中的 *** "me.jifengzhang.aidldemo.IBookManager" ***


** asInterface **

转换服务端传递过来的 IBinder 对象为客户端需要的 AIDL 接口类型的对象，这个转换过程是区分进程的，如果客户端和服务端是同一个进程，那么该方法返回的就是服务端的 Stub 对象本身，当然如果不是同一个进程则返回的是封装后的 Stub.Proxy 对象。
```java
public static me.jifengzhang.aidldemo.IBookManager asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof me.jifengzhang.aidldemo.IBookManager))) { //同一个进程
        return ((me.jifengzhang.aidldemo.IBookManager) iin);
    }
    //不同进程
    return new me.jifengzhang.aidldemo.IBookManager.Stub.Proxy(obj);
}
```

** asBinder **

该方法用于返回当前 Binder 对象

** onTransact **

该方法运行在服务端的 Binder 线程池中， 当客户端发起跨进程请求时，远程请求会通过系统底层封装后交给该方法处理。服务端通过 code 来确定客户端请求的方法是什么，从 data：Parcel 中取出方法所需要的参数，然后执行目标方法，最后将方法执行的结果（返回值）存到 replay: Parcel 中。

** Proxy#getBookList **

该方法运行在客户端，当客户端调用该方法时，首先创建该方法所需要的输入 Parcel 对象 _data, 输出 Parcel 对象 _reply 以及返回值 List，然后将参数信息写入 _data 中，接着调用 transact 方法发起远程调用 RPC ，此时客户端当前线程挂起，服务端的 onTransact 方法会被调用执行，直到 RPC 过程返回后，客户端当前线程继续执行，并从 _reply 中取出 RPC 过程的返回结果。最后将 _reply 转换成 List 返回。

** Proxy#addBook

执行过程和 getBookList 一样。

通过简单的分析描述，其实 AIDL 的实现其实很简单，方法调用的流程大概如下：
![AIDL 流程](http://7u2jwu.com1.z0.glb.clouddn.com/blog/20170503/182015065.png)

### 服务端（Service）实现
上面简单说明了 AIDL 中接口的定义，这里我们来看看服务端也就是作为服务提供者的 Service 的实现，我们可以想到的是 Service 肯定是要实现 IBookManager 定义的两个接口，剩余的应该有 Service 的基本实现。那我们来看代码：
```java
public class BookManagerService extends Service {
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "IOS"));
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
果然，代码和我们想的是一样的，这里通过 new 一个 Stub (IBookManager 的内部类，继承自 Binder)来实现 getBookList 和 addBook 两个接口方法。

### 客户端的实现
客户端也就是访问端的实现也比较简单，首先是绑定服务(BinderService)，绑定成功后将服务端返回的 Binder 对象转换为 AIDL 接口，然后就可以使用这个接口来调用 Server 端的远程方法。
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    //绑定服务
    Intent intent = new Intent(this, BookManagerService.class);
    bindService(intent, mServiceConnect, Context.BIND_AUTO_CREATE);
}

@Override
protected void onDestroy() {
    super.onDestroy();
    //解绑
    unbindService(mServiceConnect);
}

private ServiceConnection mServiceConnect = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        //将服务端传递过来的Binder对象转换为AIDL接口
        IBookManager manager = IBookManager.Stub.asInterface(service);
        try {
            List<Book> list = manager.getBookList();
            Log.i("MainActivity", "queue book list : " + list.toString());
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
};

```

### 运行结果

我们在 Manifest 中给 BookManagerService 设置 process 属性，这样 BookManagerService 和 MainActivity 就是不同的进程。

```
<service android:name=".BookManagerService" android:process=":remote"/>
```
运行结果：
```
05-04 10:10:28.622 17622-17622/me.jifengzhang.aidldemo I/MainActivity: queue book list : [[1:Android], [2:IOS]]
```

## 参考

[Gityuan](http://gityuan.com/2015/10/31/binder-prepare/)

** Android 开发艺术探索 **
