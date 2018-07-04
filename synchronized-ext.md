## 共享数据与锁
Java虚拟机的运行时数据区中的堆和方法区是所有线程共享的区域，如果多个线程需要同时使用共享的对象或类变量，则必须要正确协调它们对数据的访问。否则，程序将具有不可预测的行为。为了协调多个线程之间的共享数据访问，Java虚拟机将锁与每个对象或类关联起来。锁就像一种特权，在任何时候只有一个线程可以“拥有”它。如果一个线程想要锁定一个特定的对象或类，它会请求JVM，在线程向JVM请求锁之后(如果锁未被持有可能很快，如果锁被持有也可能稍后，也可能永远不会)，JVM将锁提供给线程。当线程不再需要锁时，它将锁返回给JVM。

## 对象锁与类锁
对象锁即类实例对象的锁。

类锁实际上是作为对象锁实现的。当JVM加载类文件时，它会创建类java.lang.Class的实例。当锁定一个类时，实际上锁定了那个类的类对象。

## Java对象的对象头
在HotSpot虚拟机中，Java对象在内存中存储的布局分为3块区域：对象头、实例数据和对齐填充。对象头包含两部分，第一部分包含对象的HashCode、分代年龄、锁标志位、线程持有的锁、偏向线程ID等数据，这部分数据的长度在32位和64位虚拟机中分别为32bit和64bit，官方称为Mark World。考虑到虚拟机的空间效率，Mark World内部的数据结构是非固定的，也就是说对象头中存储的内容是不固定的，下图展示了不同状态下，对象头中存储的内容：
![对象头](https://github.com/wind7rui/HighConcurrency/blob/master/Object-Mark-World.png)

当使用synchronized修饰方法或修饰语句块时(即获取对象锁或类锁时)，对象(类实例对象或类的类对象)的对象头中锁状态处于重量级锁，此时锁标志位为10，其余30bit用于存储指向互斥量(重量级锁)的指针，这里的指针，笔者理解为monitor对象的地址。

## Monitor
Java虚拟机中，synchronized支持的同步方法和同步语句都是使用monitor来实现的。每个对象都与一个monitor相关联，当一个线程执行到一个monitor监视下的代码块中的第一个指令时，该线程必须在引用的对象上获得一个锁，这个锁是monitor实现的。

在HotSpot虚拟机中，monitor是由ObjectMonitor实现，使用C++编写实现，具体代码在HotSpot虚拟机源码ObjectMonitor.hpp文件中。
```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0;    // 记录该线程获取锁的次数
    _waiters      = 0,
    _recursions   = 0;    // 锁的重入次数
    _object       = NULL;
    _owner        = NULL; // 指向持有ObjectMonitor对象的线程
    _WaitSet      = NULL; // 处于wait状态的线程集合
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 处于等待锁block状态的线程队列
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
当并发线程执行synchronized修饰的方法或语句块时，先进入_EntryList中，当某个线程获取到对象的monitor后，把monitor对象中的_owner变量设置为当前线程，同时monitor对象中的计数器_count加1，当前线程获取同步锁成功。

当synchronized修饰的方法或语句块中的线程调用wait()方法时，当前线程将释放持有的monitor对象，monitor对象中的_owner变量赋值为null，同时，monitor对象中的_count值减1，然后当前线程进入_WaitSet集合中等待被唤醒。

一个线程可以多次锁定同一个对象。对于每个对象，JVM维护对象被锁定的次数的计数。未加锁的对象的计数为零。当线程第一次获得锁时，计数将增加到1。每次线程获取同一个对象上的锁时，都会增加一个计数。每次线程释放锁时，计数将被递减。当计数达到0时，锁被释放，此时其它线程可以继续请求获取锁。

下图展示获取锁和释放锁monitor中数据变化：
![monitor](https://github.com/wind7rui/HighConcurrency/blob/master/monitor.png)

