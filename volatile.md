# volatile

volatile的实现原理包括JMM中的原子性和可见性规则，和JSR-133对volatile的有序性语义增强



## JMM

JMM是对线程和主存关系的一种抽象，每个线程都有自己的工作内存用于操作变量。JMM的大多数规则都是关于变量如何在线程和主存之间的转移。

### Atomicity

对所有变量（在32位虚拟机除了 long 和 double，包括对类的引用、volatile修饰的long and double）的访问和修改都是原子的。



### Visibility

可见性表示线程对变量的修改对于其他线程可见，仅存在以下几种情况

* 线程释放同步锁和另一个线程获取相同的同步锁(本质就是锁释放会把修改的变量刷新绘主存，锁获取会重新从主存中加载)
* 线程对volatile修饰的变量的修改回立即刷新回主存，其他线程访问volatile修饰的变量每次都会重新从主存中加载
* 线程第一次访问变量时，看到的要么是初始化的值，要么是其他线程修改后的值
* 线程结束后会把修改的变量刷新回主存



### Ordering

* 单线程，编译器和处理器按照 as-if-serial 保证执行结果不会改变
* 多线程，并发执行未同步(加锁)的代码块，almost anything can happen



## JSR-133

JSR-133进一步加强了 volatile 的语义，同时synchronized的monitorenter和monitorexit也同样禁止重排序。

* 禁止编译重排序

  volatile变量读/写 与 其他volatile变量或普通变量的读/写 禁止重排序

* 禁止指令重排序

  通过插入内存屏障防止CPU对指令的重排序