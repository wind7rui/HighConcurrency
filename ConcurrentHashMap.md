## 要点解说
ConcurrentHashMap是支持并发操作、线程安全的HashMap。JDK1.7之前(包含7)的版本中，其底层使用数组+链表的结构，并使用segment实现并发安全控制，其内部实现还是基于ReentrantLock非公平锁。JDK1.8版本中底层使用数组+链表+红黑树的结构，并使用CAS+synchronized实现并发安全控制。

### JDK1.7之前(包含7)的版本
JDK1.7之前(包含7)的版本中，每个ConcurrentHashMap中包含一个Segment数组；每个Segment继承自ReentrantLock，其内部有一个HashEntry数组；每个HashEntry内部包含存储的key、value、hash和下一个HashEntry引用。ConcurrentHashMap底层数据结构如下图所示。

![](https://github.com/wind7rui/HighConcurrency/blob/master/ConcurrentHashMap1.7.png)

### JDK1.8版本
从JDK1.8版本开始，取消了Segment分段锁，每个ConcurrentHashMap中包含一个Node数组；每个Node内部包含存储的key、value、hash和next下一个Node引用；next构成了链表，当链表上的元素个数大于8时，就将其转化为一颗红黑树，提高检索效率。ConcurrentHashMap底层数据结构如下图所示。

## 源码分析

### JDK1.7之前(包含7)的版本

构造方法

put

get

### JDK1.8版本

构造方法

put

get
