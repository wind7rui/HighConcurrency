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
3. unlock()：释放读操作锁，同时唤醒其它等待获取该锁的线程；
4. tryLock()：尝试获取读操作锁，如果写入锁没有被其它线程持有，则立即获取读取锁并返回true值，否则立即返回false值；
5. tryLock(long timeout, TimeUnit unit)：在指定的等待时间内尝试获取读操作锁，如果写入锁没有被其它线程持有，并且当前线程未被中断，则立即获取读取锁并返回true值，否则当前线程阻塞直到等待时间结束并返回false。

### WriteLock类
1. ReentrantReadWriteLock.WriteLock(ReentrantReadWriteLock lock)：使用ReentrantReadWriteLock创建WriteLock对象；
2. lock()：当前线程获取写入锁时，如果其它线程既没有持有读取锁也没有持有写入锁，则可以获取写入锁并立即返回，并将写入锁持有计数设置为1；如果当前线程已经持有写入锁，则写入锁计数增加1，该方法立即返回；如果锁被其它线程持有，当前线程阻塞直到获取写入锁；
3. tryLock()：当前线程获取写入锁时，如果其它线程既没有持有读取锁也没有持有写入锁，则可以获取写入锁并立即返回，并将写入锁持有计数设置为1；如果当前线程已经持有写入锁，则写入锁计数增加1，该方法立即返回；如果锁被其它线程持有，立即返回false值；
4. tryLock(long timeout, TimeUnit unit)：在指定的等待时间内尝试获取写操作锁，如果其它线程既没有持有读取锁也没有持有写入锁，并且当前线程未被中断，则可以获取写入锁并立即返回，并将写入锁持有计数设置为1；如果当前线程已经持有此锁，则将持有计数加1，该方法将返回true值；否则当前线程阻塞直到等待时间结束并返回false；
5. unlock()：如果当前线程持有此锁，则将持有计数减1；如果持有计数等于0，则释放该锁，同时唤醒其它等待获取该锁的线程；如果当前线程不是此锁的持有者，则抛出 IllegalMonitorStateException。

## 源码解析

