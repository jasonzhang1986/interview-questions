synchronized 关键字，它包括两种用法：**synchronized 方法** 和 **synchronized 块**; synchronized 关键字可以作为函数的修饰符，也可作为函数内的语句，也就是平时说的同步方法和同步语句块。如果再细的分类，
synchronized 可作用于 instance 变量、object reference（对象引用）、static 函数和 class literals (类名称字面常量，比如 **TestClass.class** )身上。

  无论 synchronized 关键字加在方法上还是对象上，它取得的锁都是对象，而不是把一段代码或函数当作锁――而且同步方法很可能还会被其他线程的对象访问。

  假设 P1、P2 是同一个类的不同对象，这个类中定义了以下几种情况的同步块或同步方法，P1、P2 就都可以调用它们。

  把 synchronized 当作函数修饰符时，示例代码如下：
  ```Java
  public synchronized void methodA() {

  //….

  }
  ```

  这也就是同步方法，那这时 synchronized 锁定的是哪个对象呢？它锁定的是调用这个同步方法对象。也就是说，当一个对象 P1 在不同的线程中执行这个同步方法时，它们之间会形成互斥，达到同步的效果。但是这个对象所属的 Class 所产生的另一对象 P2 却可以任意调用这个被加了 synchronized 关键字的方法。

  上边的示例代码等同于如下代码：
  ```Java
  public void methodA() {
      synchronized (this)//(1) {

         //…..

     }
  }
  ```
  (1)处的 this 指的是什么呢？它指的就是调用这个方法的对象，如 P1。可见同步方法实质是将 synchronized 作用于 object reference（那个拿到了 P1 对象锁的线程），才可以调用 P1 的同步方法，而对 P2 而言，P1 这个锁与它毫不相干，程序也可能在这种情形下摆脱同步机制的控制，造成数据混乱。
