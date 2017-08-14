## 一、正常情况下 Activity 的生命周期
 ![LifeCycle](ActivityLifeCycle.png)

* **onCreate 方法**，Activity 被启动时的第一个方法，表示 Activity 正在被创建，在这个方法中一般做一些初始化的工作，比如调用 setContentView 方法加载界面布局，并且可以做一些初始化数据的工作：声明UI元素，定义成员变量，配置 UI 等。

    **onCreate 尽量少做一些不必要的操作，避免 Activity 启动太久，使用户半天看不到 Activity**

* **onStart 方法**，表示 Activity 正在被启动，其实这时 Activity 已经可见了，正如上图所表示的 Visible ,但并不能被用户所见，我们可以这样理解，Activity 现在运行到了后台，还没有准备好到前台和用户交互。

* **onResume 方法**，此时 Activity 来到前台和用户进行交互，用户真正的看到了 Activity 。

    **往往程序在 onCreate 执行完成之后会迅速调用 onStart 方法和 onResume 方法。**

* **onPause 方法**,表示 Activity 正在停止，该状态下，activity 的部分被另外一个 activity 所遮盖：另外的 activity 来到前台，但是半透明的，不会覆盖整个屏幕。被暂停的 activity 不再接受用户的输入且不再执行任何代码。

* **onStop 方法**，该状态下, activity 完全被隐藏，对用户不可见。可以认为是在后台。当 stopped, activity 实例与它的所有状态信息（如成员变量等）都会被保留，但 activity 不能执行任何代码。

    ** 整个生命周期方法中，只有 onResume, onPause, onStop 方法是“静态”的，既可以存在较长的时间的。其它状态 ( Created 与 Started )都是短暂的，系统快速的执行那些回调函数并通过执行下一阶段的回调函数移动到下一个状态。也就是说，在系统调用 onCreate (), 之后会迅速调用 onStart (), 之后再迅速执行 onResume()。**

* **onRestart 方法**，Activity 从不可见又变为可见时会调用。此时调用顺序为: onRestart -> onStart -> onResume

* **onDestroy 方法**，表示 Activity 即将被销毁，当收到需要将该 activity 彻底移除的信号时，系统会调用这个方法。
    - 大多数 app 并不需要实现这个方法，因为局部类的 references 会随着 activity 的销毁而销毁，并且我们的 activity 应该在 onPaus ()与 onStop ()中执行清除 activity 资源的操作。然而，如果 activity 含有在 onCreate 调用时创建的后台线程，或者是其他有可能导致内存泄漏的资源，则应该在 onDestroy ()时进行资源清理，杀死后台线程。
    - 除非程序在 onCreate ()方法里面就调用了 finish ()方法，系统通常是在执行了 onPause ()与 onStop () 之后再调用 onDestroy () 。在某些情况下，例如我们的 activity 只是做了一个临时的逻辑跳转的功能，它只是用来决定跳转到哪一个 activity ，这样的话，需要在 onCreate 里面调用finish方法，这样系统会直接调用 onDestory ，跳过生命周期中的其他方法。

## 二、异常情况下的生命周期分析

* **资源相关的系统配置发生改变导致 Activity 被杀死并重新创建**

    当系统配置发生改变后，如从横屏手机切换到了竖屏，Activity 会被销毁，其 onPause, onStop, onDestroy 方法均会被调用，同时由于是在异常情况下被终止的，系统会调用 onSavedInstanceState 来保存当前 Activity 的状态(正常情况下不会调用此方法)，这个方法的调用时机是在 onStop 之前，当 Activity 重新被创建后，系统调用会调用
    ```Java
    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        super.onRestoreInstanceState(savedInstanceState);
    }
    ```
    方法，并且把 Activity 销毁时 onSavedInstanceState 方法所保存的 Bundle 对象作为参数同时传递给 onRestoreInstanceState 和 onCreate 方法，onRestoreInstanceState 是在 onStart 之后调用。

    *我们知道，当Activity被异常终止后被恢复时，系统会自动的帮我们恢复数据和一些状态，如文本框用户输入的数据，listview 的滚动位置等，关于保存和恢复 View 的层次结构和数据，系统的工作流程是这样的，首先 Activity 被意外终止时，Activity 会调用 onSavedInstanceState 去保存数据，然后 Activity 会委托 Window 去保存数据，接着 Window 再委托它上面的顶级容器去保存数据，顶层容器是一个 ViewGroup，一般来说它很可能是一个 DecorView 。最后顶层容器再去一一通知它的子元素来保存数据，这样整个数据保存过程就完成了，可以发现，这是一种典型的委托思想，上层委托下层，父容器委托子容器去处理一些事情。*

* **资源内存不足导致低优先级的Activity被杀死**

    Activity按照优先级我们可以分为以下的三种：
    - 前台Activity—正在和用户交互的Activity，优先级最高。
    - 可见但非前台Activity,如处于onPause状态的Activity，Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户直接交互。
    - 后台Activity—已经被暂停的Activity,比如执行了onStop方法，优先级最低。

  当系统内存不足时，系统就会按照上述的优先级顺序选择杀死 Activity 所在的进程，并在后续通过 onSaveInstanceState 缓存数据和 onRestoreInstanceState 恢复数据。

 ***如果一个进程中没有四大组件在执行，那么这个进程将很快被杀死，因此，一些后台工作不适合脱离了四大组件工作，比较好的方法是将后台工作放入Service中从而保证进程有一定的优先级，这样就不会轻易的被系统杀死。***

 ***系统只恢复那些被开发者指定过id的控件，如果没有为控件指定id,则系统就无法恢复了。***
