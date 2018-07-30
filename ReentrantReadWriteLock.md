## 要点解说
ReentrantLock在并发情况下只允许单个线程执行受保护的代码，而在大部分应用中都是读多写少，所以，如果使用ReentrantLock实现这种对共享数据的并发访问控制，将严重影响整体的性能。ReentrantReadWriteLock中提供的读取锁(ReadLock)可以实现并发访问下的多读，写入锁(WriteLock)可以实现每次只允许一个写操作。但是，在它的实现中，读、写是互斥的，即允许多读，但是不允许读写和写读同时发生。

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
首先，看一下ReentrantReadWriteLock类有哪些属性和构造函数，具体代码如下。
```
    public class ReentrantReadWriteLock
            implements ReadWriteLock, java.io.Serializable {

        //ReadLock是ReentrantReadWriteLock中的静态内部类，它是读取锁的实现
        private final ReentrantReadWriteLock.ReadLock readerLock;

        //WriteLock是ReentrantReadWriteLock中的静态内部类，它是写入锁的实现
        private final ReentrantReadWriteLock.WriteLock writerLock;

        //Sync是ReentrantReadWriteLock中的静态内部类，它继承了AQS
        //它是读写锁实现的重点，后面深入分析
        final Sync sync;

        //默认使用非公平策略创建对象
        public ReentrantReadWriteLock() {
            this(false);
        }

        //根据指定策略参数创建对象
        public ReentrantReadWriteLock(boolean fair) {
            //FairSync和NonfairSync都继承自Sync，它们主要提供了对读写是否需要被阻塞的检查方法
            sync = fair ? new FairSync() : new NonfairSync();
            readerLock = new ReadLock(this);
            writerLock = new WriteLock(this);
        }
    }
```
上面的代码中介绍到Sync类是重点代码，这里先看它的部分重点源码。
```
    //继承了AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {
        //常量值
        static final int SHARED_SHIFT   = 16;

        //左移16位后，二进制值是10000000000000000，十进制值是65536
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);

        //左移16位后再减一，十进制值是65535
        //这个常量值用于标识最多支持65535个递归写入锁或65535个读取锁
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;

        //左移16位后再减一，二进制值是1111111111111111
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        //用于计算持有读取锁的线程数
        static int sharedCount(int c) {
            //无符号右移动16位
            //如果c是32位，无符号右移后，得到是高16位的值
            return c >>> SHARED_SHIFT; 
        }
        
        //用于计算写入锁的重入次数
        static int exclusiveCount(int c) {
            //如果c是32位，和1111111111111111做&运算，得到的低16位的值
            return c & EXCLUSIVE_MASK; 
        }

        //用于每个线程持有读取锁的计数
        static final class HoldCounter {
            //每个线程持有读取锁的计数
            int count = 0;

            //当前持有读取锁的线程ID
            //这里使用线程ID而没有使用引用，避免垃圾收集器保留，导致无法回收
            final long tid = Thread.currentThread().getId();
        }

        //通过ThreadLocal维护每个线程的HoldCounter
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            //这里重写了ThreadLocal的initialValue方法
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }

        //当前线程持有的可重入读取锁的数量，仅在构造方法和readObject方法中被初始化
        //当持有锁的数量为0时，移除此对象
        private transient ThreadLocalHoldCounter readHolds;

        //成功获取读取锁的最近一个线程的计数
        private transient HoldCounter cachedHoldCounter;

        //第一个获得读锁的线程
        private transient Thread firstReader = null;
        //第一个获得读锁的线程持有读取锁的次数
        private transient int firstReaderHoldCount;

        Sync() {
            //构建每个线程的HoldCounter
            readHolds = new ThreadLocalHoldCounter();
            setState(getState()); // ensures visibility of readHolds
        }
    }
```
Sync继承自AQS，Sync使用AQS中的state属性来代表锁的状态，这个state二进制值被设计成32位，其中高16位用作读取锁，低16位用作写入锁。所以，如果要计算持有读取锁的线程数，只要将state二进制值无符号右移动16位；如果要计算写入锁的重入次数，只要将state二进制值和1111111111111111做&运算。

### ReadLock源码解析
下面开始分析读取锁ReadLock的实现原理，先看一下它的构造函数源码。
```
    public static class ReadLock implements Lock, java.io.Serializable {
        private final Sync sync;

        //通过ReentrantReadWriteLock对象构建ReadLock
        protected ReadLock(ReentrantReadWriteLock lock) {
            //在ReentrantReadWriteLock构造函数中会根据fair参数值选择FairSync或NonfairSync创建不同的对象
            //所以，这里赋值给sync的可能是FairSync类的对象，也可能是NonfairSync类的对象
            sync = lock.sync;
        }
    }
```
FairSync和NonfairSync都继承自Sync，不同点是各自实现了对读写是否需要被阻塞的检查方法，这里不做深入分析。

对于lock()方法，如果写入锁没有被其它线程持有，则立即获取读取锁，否则当前线程阻塞直到获取读取锁，具体代码如下。
```
        public void lock() {
            //调用AQS的acquireShared方法
            sync.acquireShared(1);
        }

        //AQS的acquireShared方法
        public final void acquireShared(int arg) {
            //尝试获取读取锁
            //tryAcquireShared方法返回值小于0，则获取失败
            if (tryAcquireShared(arg) < 0)
                //AQS中的方法，用于排队尝试再次获取读取锁
                doAcquireShared(arg);
        }
        
        //ReentrantReadWriteLock类的tryAcquireShared方法
        protected final int tryAcquireShared(int unused) {
            //获取当前线程
            Thread current = Thread.currentThread();
            //获取当前AQS中state值
            int c = getState();
            //使用exclusiveCount方法计算写入锁是否被持有
            //如果exclusiveCount(c)结果不等于0，即写入锁被持有，并且持有写入锁的线程不是当前线程
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                //返回-1
                return -1;
            //使用sharedCount方法计算读取锁被持有的线程数
            int r = sharedCount(c);
            //如果当先线程不需要被阻塞，并且持有读取锁的线程数没有超过最大值，并且使用CAS更新读取锁线程数量成功。
            //readerShouldBlock方法用于判断当先线程是否需要被阻塞，它在FairSync和NonfairSync中的实现各不同，
            //FairSync中实现是如果有其它线程在当前线程之前等待获取读取锁，则当前线程应该被阻塞并返回true，否则返回false；
            //NonfairSync中实现是如果等待队列的第一个节点的线程等待获取写入锁，则当前线程应该被阻塞并返回true，否则返回false；
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //如果读取锁被持有的线程数等于0
                if (r == 0) {
                    //则表示当前线程是第一个获取读取锁的
                    firstReader = current;
                    //第一个获得读锁的线程持有读取锁的次数赋值为1
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //如果读取锁被持有的线程数不等于0，并且当前线程是第一个获取读取锁的
                    //则将持有读取锁的次数加1
                    firstReaderHoldCount++;
                } else {
                    //将当先线程持有读取锁的次数信息，放入线程本地变量中
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId())
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                //成功获取读取锁，返回1
                return 1;
            }
            //如果当前线程需要被阻塞，或持有读取锁的线程数超过最大值，或使用CAS更新读取锁线程数量失败
            //通过自旋的方式解决这三种获取锁失败的情况
            return fullTryAcquireShared(current);
        }

        //ReentrantReadWriteLock类的fullTryAcquireShared方法
        final int fullTryAcquireShared(Thread current) {
            
            HoldCounter rh = null;
            //自旋
            for (;;) {
                //当前锁状态值
                int c = getState();
                //如果写入锁被线程持有
                if (exclusiveCount(c) != 0) {
                    //并且写入锁的持有者不是当前线程，则返回-1，获取锁失败
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                //如果当前线程应该被阻塞
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        //如果最近一次获取读取锁的线程或从上下文中获取的当前线程，持有的锁记录数等于0，
                        //则返回-1获取读取锁失败
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != current.getId()) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                //如果持有读取锁的线程数等于最大值
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //如果使用CAS更新读取锁线程数量成功
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    //更新线程持有的锁次数
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
        
        //AQS中的doAcquireShared方法
        private void doAcquireShared(int arg) {
            //根据当前线程创建一个共享模式的Node节点
            //并把这个节点添加到等待队列的尾部
            //AQS等待队列不熟悉的可以查看AQS深入解析的内容
            //addWaiter方法的解析在其它篇幅已经分析过，这里不再深入分析
            final Node node = addWaiter(Node.SHARED);
            boolean failed = true;
            try {
                boolean interrupted = false;
                //通过自旋尝试获取读取锁
                for (;;) {
                    //获取新建节点的前驱节点
                    final Node p = node.predecessor();
                    //如果前驱节点是头结点
                    if (p == head) {
                        //尝试获取读取锁
                        int r = tryAcquireShared(arg);
                        //获取到读取锁
                        if (r >= 0) {
                            //将前驱节点从等待队列中释放
                            //同时使用LockSupport.unpark方法唤醒前驱节点的后继节点中的线程
                            setHeadAndPropagate(node, r);
                            p.next = null; // help GC
                            if (interrupted)
                                selfInterrupt();
                            failed = false;
                            return;
                        }
                    }
                    //当前节点的前驱节点不是头结点，或不可以获取到锁
                    //shouldParkAfterFailedAcquire方法检查当前节点在获取锁失败后是否要被阻塞
                    //如果shouldParkAfterFailedAcquire方法执行结果是当前节点线程需要被阻塞，则执行parkAndCheckInterrupt方法阻塞当前线程
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }
```
对于unlock()方法用于释放当前线程持有的读取锁，具体实现是先修改线程持有的读取锁的计数数量，然后通过自旋方式使用CAS修改锁状态高16位值为当前值减1，最后如果锁状态值等于0，即既没有写入锁，也没有读取锁被线程持有，则唤醒等待队列中等待获取锁的线程。具体源码分析如下。
```
        //ReadLock类中的unlock方法
        public  void unlock() {
            //调用AQS中的releaseShared方法
            sync.releaseShared(1);
        }
        
        //AQS中的releaseShared方法
        public final boolean releaseShared(int arg) {
            //这里调用了ReentrantReadWriteLock类的tryReleaseShared方法
            //在这个方法中做了如下两件事：
            //1.修改当前线程持有的读取锁的计数数量，计数数量减一
            //2.修改锁状态值的高16位值(读取锁的值)，值减一，并且返回锁状态值是否等于0
            if (tryReleaseShared(arg)) {
                //如果锁状态值等于0，即既没有写入锁，也没有读取锁被线程持有，执行doReleaseShared
                //这里调用的是AQS中的doReleaseShared方法
                doReleaseShared();
                return true;
            }
            return false;
        }
        
        //ReentrantReadWriteLock类的tryReleaseShared方法
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            // 如果当前线程是第一个获取读取锁的线程
            if (firstReader == current) {
                // 第一个获取读取锁的线程持有的读取锁的次数等于1
                if (firstReaderHoldCount == 1)
                    //要释放掉，所以置为null
                    firstReader = null;
                else
                    //否则，直接将线程持有的读取锁的次数减一
                    firstReaderHoldCount--;
            } else{
                //最近一次成功获取读取锁的线程和计数信息
                HoldCounter rh = cachedHoldCounter;
                //如果取得的rh==null或它里面记录的不是当前线程的信息，
                //则直接从上下文中获取当前线程的线程和计数信息
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                //当前线程持有读取锁的次数
                int count = rh.count;
                //如果次数小于等于1，则从上下文中移除
                if (count <= 1) {
                    readHolds.remove();
                    //如果次数小于等于0，则抛出异常，因为不符合是否锁的条件
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                //将线程持有读取锁的次数减一
                --rh.count;
            }
            for (;;) {
                //获取锁状态值
                int c = getState();
                //这里转换成二进制计算，是将高16位减1
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    //如果既没有写入锁，也没有读取锁被线程持有，则返回true
                    return nextc == 0;
            }
        }
        
        //AQS中的doReleaseShared方法
        private void doReleaseShared() {
            for (;;) {
                //等待队列的头结点
                Node h = head;
                //如果头结点存在，并且有后继节点
                if (h != null && h != tail) {
                    int ws = h.waitStatus;
                    //如果head指向的节点状态可以被唤醒
                    if (ws == Node.SIGNAL) {
                        //使用CAS更新节点的等待状态
                        //如果不成功则继续循环，通过循环来保证更新成功
                        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                            continue;            // loop to recheck cases
                        //更新成功，则唤醒节点对应的线程
                        unparkSuccessor(h);
                    }
                    else if (ws == 0 &&
                             !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                        continue;                // loop on failed CAS
                }
                if (h == head)                   // loop if head changed
                    break;
            }
        }
```
