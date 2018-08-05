## 要点解说
支持并发操作、线程安全的HashMap。JDK1.7之前(包含7)的版本中，使用Segment分段锁实现并发安全，其内部实现还是基于ReentrantLock非公平锁。

## 源码分析
