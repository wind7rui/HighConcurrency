## 要点解说
ConcurrentHashMap是支持并发操作、线程安全的HashMap。JDK1.7之前(包含7)的版本中，其底层使用数组+链表的结构，并使用segment实现并发安全控制，其内部实现还是基于ReentrantLock非公平锁。JDK1.8版本中底层使用数组+链表+红黑树的结构，并使用synchronized实现并发安全控制。

JDK1.7之前(包含7)的版本中，ConcurrentHashMap底层数据结构如下图所示。
![](https://github.com/wind7rui/HighConcurrency/blob/master/ConcurrentHashMap1.7.png)

JDK1.8版本中，ConcurrentHashMap底层数据结构如下图所示。

## 源码分析
