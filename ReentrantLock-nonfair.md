## 要点解说
ReentrantLock是一个可重入的互斥锁，它不但具有synchronized实现的同步方法和同步代码块的基本行为和语义，而且具备很强的扩展性。ReentrantLock提供了公平锁和非公平锁两种实现，在默认情况下构造的ReentrantLock实例是非公平锁，可以在创建ReentrantLock实例的时候通过指定公平策略参数来指定是使用公平锁还是非公平锁。本篇将基于JDK7深入源码解析非公平锁的实现原理。

## 实例演示
下面是使用ReentrantLock非公平锁的典型代码。
```
class X {
  private final ReentrantLock lock = new ReentrantLock();

  public void m() {
    lock.lock();
    try {
      // do some thing
    } finally {
      lock.unlock()
    }
  }
}
```

## 方法解析
1.ReentrantLock()创建一个非公平锁ReentrantLock实例。 
2.ReentrantLock(boolean fair)根据公平策略fair参数创建ReentrantLock实例。 
3.lock()获取锁。 
4.unlock()释放锁。 
5.newCondition()返回与此ReentrantLock实例一起使用的Condition的实例。 
6.getHoldCount()获取当前线程持有此锁的次数。 
7.getQueueLength()返回正在等待获取此锁的线程数。 
8.getWaitQueueLength(Condition condition)返回等待与此锁相关的给定条件的线程数。 
9.hasQueuedThread(Thread thread)返回指定线程是否正在等待获取此锁。 
10.hasQueuedThreads()返回是否有线程正在等待获取此锁。 
11.hasWaiters(Condition condition)返回是否有线程正在等待与此锁有关的给定条件。 
12.isFair()返回锁是否是公平锁。 
13.isHeldByCurrentThread()返回当前线程是否持有此锁。 
14.tryLock()尝试获取锁，仅在调用时锁未被其它线程持有时才可以获取该锁。 
15.tryLock(long timeout, TimeUnit unit)尝试获取锁，如果锁在指定等待时间内没有被另一个线程持有，并且当前线程未被中断，则可以获取该锁
