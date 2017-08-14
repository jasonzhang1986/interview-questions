
### Android相关
* [Activity 正常和异常情况下的生命周期](/Android/ActivityLifeCycle.md)
* [Activity 的四种启动模式](http://blog.csdn.net/zhangjg_blog/article/details/10923643)
* 画出 Android 的大体架构图
* [app 如何保证后台服务不被杀死](http://dev.qq.com/topic/57ac4a0ea374c75371c08ce8)
* [Activity 与 Fragment 生命周期有何联系](http://www.cnblogs.com/purediy/p/3276545.html)
* [Activity 与 Fragment 之间如何进行通信？](http://blog.csdn.net/u012702547/article/details/49786417)
* [IntentService 比 Service 好在哪](http://blog.qiji.tech/archives/2693)
* Thread 和 HandlerThread 区别
* 关于< include >< merge >< stub >三者的使用场景
* 对 Android 消息机制的理解( Binder、Handler )
* Android 上的 Inter-Process-Communication 跨进程通信时如何工作的；
* 如何优化一个 ListView ?
* listview图片加载错乱的原理和解决方案
* RecyclerView与ListView 的区别，缓存机制的不同，性能
* 哪些情况会导致 OOM ？
* android什么情况下会发生内存泄露
* 造成 ANR 的原因？
* 如何监测内存泄露？有哪些工具？
* 用 leak 工具监测内存泄露的原理是什么？
* 界面卡顿的原因有哪些？
* 图片加载原理
* [如何优雅的展示 Bitmap 大图](http://blog.csdn.net/guolin_blog/article/details/9316683)
* SP 是进程同步的吗?有什么方法做到同步
* BroadcastReceiver，LocalBroadcastReceiver 区别
* 广播（动态注册和静态注册区别，有序广播和标准广播）
* service生命周期
* 介绍下 SurfaceView、TextureView
* android 事件传递机制
* 多线程（关于AsyncTask缺陷引发的思考）
* App启动流程，从点击桌面开始
* 谈谈 ClassLoader
* 动态布局
* 热修复,插件化
* HashMap 源码, SpareArray 原理
* 性能优化,怎么保证应用启动不卡顿
* 怎么去除重复代码
* 动态加载
* Https请求慢的解决办法(DNS，携带数据，直接访问IP)
* GC回收策略
* 描述清点击 Android Studio 的 build 按钮后发生了什么
* 大体说清一个应用程序安装到手机上时发生了什么
* 对 Dalvik、ART 虚拟机有基本的了解
* App 是如何沙箱化，为什么要这么做
* 权限管理系统（底层的权限是如何进行 grant 的）
* 进程和 Application 的生命周期
* 系统启动流程 Zygote进程 –> SystemServer进程 –> 各种系统服务 –> 应用进程
* MVP
* 网络请求缓存处理，okhttp如何处理网络缓存的
* 模块化实现（好处，原因）
* 统计启动时长,标准
* 如何保持应用的稳定性
* 数据库数据迁移问题
* 设计模式相关（例如Android中哪里使用了观察者模式，单例模式相关）
* 微信的聊天数据在本地都是加密处理的（防止root了被破解），设计一个类似的本地数据存储系统
* x个苹果，一天只能吃一个、两个、或者三个，问多少天可以吃完
* Android相关你最擅长哪一块
* TCP与UDP区别与应用（三次握手和四次挥手）涉及到部分细节（如client如何确定自己发送的消息被server收到） HTTP相关 提到过Websocket 问了WebSocket相关以及与socket的区别
* 是否熟悉Android jni开发，jni如何调用java层代码
* 进程间通信的方式
* java注解
* 计算一个view的嵌套层级
* 项目组件化的理解
* 基于自身工作经验和计算机相关知识，给出 移动端地图局部加载 瓦片大小的像素大小估值
* 多线程断点续传原理
* Android系统为什么会设计ContentProvider，进程共享和线程安全问题
* jvm相关
* Android相关优化（如内存优化、网络优化、布局优化、电量优化、业务优化）
* EventBus实现原理
* 线程挂起，休眠，释放资源相关，唤醒，线程同步，数据传递，问了很多线程的问题，问了20分钟大概
* static synchronized 方法的多线程访问和作用，同一个类里面两个synchronized方法，两个线程同时访问的问题
* 内部类和静态内部类和匿名内部类，以及项目中的应用
* 泛型是什么以及在项目中的应用
* handler发消息给子线程，looper怎么启动
* down、move、up事件的传递
* activity栈
* 封装view的时候怎么知道view的大小
* intent-filter
* arraylist和linkedlist的区别，以及应用场景
* 怎么启动service，service和activity怎么进行数据交互
* 下拉状态栏是不是影响activity的生命周期，如果在onStop的时候做了网络请求，onResume的时候怎么恢复
* view渲染
* RxJava中map和flatmap操作符的区别及底层实现
* Retrofit使用的注解是哪种注解？以及，注解的底层实现是怎样的
* 视频加密传输
* OSGI


### Java相关
* Java是值传递还是引用传递
* final和static关键字的区别
* static修饰的方法可以被子类重写吗？为什么？
* HashMap HashSet HashTable的区别？
* 如何让HashMap可以线程安全？
* 序列化是什么?如何实现它?
* 垃圾收集器是什么?它是如何工作的
* 深拷贝和浅拷贝的区别
* clone()的默认实现是深拷贝还是浅拷贝?如何让clone()实现深拷贝？
* 动态代理和静态代理
* JVM的内存分布及垃圾回收机制
* Java有哪几种创建新线程的方法及区别
* ThreadLocal的理解
* Java多线程之间如何通信
* 线程池的实现机制
* Integer类对int的优化
* 单例模式有哪些实现方式
* 通过静态内部类实现单例模式有哪些优点
* synchronized volatile关键字有什么区别？以及还有哪些同样功能的关键字
* [synchronize用法](/Java/Synchronized.md)
* 操作系统进程间通信有哪些方法
* 谈谈对Socket的理解
* 不同架构的机器有何不同（如x86等）
* TCP/UDP比较
* 什么时候会发生死锁
* 操作系统层面上，线程可以加哪些锁
* 栈在系统中的方向是怎样的？为什么？
* Java 中的类型转换
* 修饰符transient和volatile的作用是什么？
* https相关，如何验证证书的合法性，https中哪里用了对称加密，哪里用了非对称加密，对加密算法（如RSA）等是否有了解
* LRUCache原理
* HashMap实现原理，ConcurrentHashMap 的实现原理

### 算法题
* 求二叉树第n层节点数
* 两个有序链表合并
* 求无序数组中的中位数
* 二叉树深度算法
* 二叉树的前序、中序、后序遍历
* 二分法查找
