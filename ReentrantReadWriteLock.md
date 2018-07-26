## 要点解说


## 实例演示
```
public class ReentrantReadWriteLockExample {
    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private final Lock readLock = readWriteLock.readLock();
    private final Lock writeLock = readWriteLock.writeLock();

    public String get(String key) {
        readLock.lock();
        try {
            //省略读取代码
            return "xxx";
        } finally {
            readLock.unlock();
        }
    }

    public void put(String key, String value) {
        writeLock.lock();
        try {
            //省略写入操作
        } finally {
            writeLock.unlock();
        }
    }
}
```

## 方法解析
ReentrantReadWriteLock类中重要的还是在它内部实现的ReadLock类和WriteLock类的方法，本篇将重点分析它们俩提供的常用方法。

### ReentrantReadWriteLock类
1. ReentrantReadWriteLock()：使用非公平策略创建一个ReentrantReadWriteLock对象；
2. ReentrantReadWriteLock(boolean fair)：根据指定的策略参数fair创建一个ReentrantReadWriteLock对象；
3. readLock()：返回用于读取操作的锁；
4. writeLock()： 返回用于写入操作的锁。

### ReadLock类
1. ReentrantReadWriteLock.ReadLock(ReentrantReadWriteLock lock)：使用ReentrantReadWriteLock创建ReadLock对象；
2. lock()：获取读操作锁，如果写入锁没有被其它线程持有，则立即获取读取锁，否则当前线程阻塞直到获取读取锁；
3. unlock()：释放读操作锁；
4. tryLock()：尝试获取读操作锁，如果写入锁没有被其它线程持有，则立即获取读取锁并返回true值，否则当前线程阻塞直到获取读取锁；
5. tryLock(long timeout, TimeUnit unit)：在指定的等待时间内尝试获取读操作锁，如果写入锁没有被其它线程持有，则立即获取读取锁并返回true值，否则当前线程阻塞直到等待时间结束并返回false；

### WriteLock类

## 源码解析

