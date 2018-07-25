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
1. ReentrantLock()创建一个非公平锁ReentrantLock实例；  
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
15.tryLock(long timeout, TimeUnit unit)尝试获取锁，如果锁在指定等待时间内没有被另一个线程持有，并且当前线程未被中断，则可以获取该锁。

## 源码解析
从上面的方法解析可以看到它有很多方法，本文将重点深入分析lock()和unlock()方法。首先，从构造函数开始。
```
    //默认构造方法创建的是非公平锁
    public ReentrantLock() {
        //构造非公平锁
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        //根据公平策略构造公平锁或非公平锁
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
从上面的代码可以看到，默认构造函数使用NonfairSync创建的是非公平锁，当然也可以指定公平策略参数fair为false创建非公平锁。NonfairSync是ReentrantLock中的静态内部类，它继承了Sync，而Sync是ReentrantLock中的抽象静态内部类，Sync又继承自AbstractQueuedSynchronizer，分析到这里可以看到，ReentrantLock的具体实现使用了AQS。
```
    abstract static class Sync extends AbstractQueuedSynchronizer {
        //此处省略内部代码，后面具体分析
    }

    static final class NonfairSync extends Sync{
        //此处省略内部代码，后面具体分析
    }
```
当调用lock()方法获取锁时，具体实现代码如下。
```
    //ReentrantLock类的lock方法
    public void lock() {
        //此时的sync是NonfairSync的实例对象，所以执行sync.lock()将执行NonfairSync类的lock()方法
        sync.lock();
    }
    
    //NonfairSync类的lock()方法
    final void lock() {
        //通过CAS设置AQS中state的值
        //如果此时state值等于0，也就是锁没有被其它线程持有，则返回true
        if (compareAndSetState(0, 1))
            //设置独占锁的当前所有者为当前线程
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //否则，尝试获取独占锁
            //acquire方法继承自AbstractQueuedSynchronizer
            acquire(1);
    }
    
    //AbstractQueuedSynchronizer类的acquire方法
    public final void acquire(int arg) {
        //调用tryAcquire方法尝试获取锁，如果成功则返回，否则先执行addWaiter方法，再执行acquireQueued方法
        //因为公平锁和非公平锁对锁持有的实现不同，所以这里的tryAcquire使用的是NonfairSync类中的实现
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //中断当前线程
            selfInterrupt();
    }
    
    //NonfairSync类的tryAcquire方法
    protected final boolean tryAcquire(int acquires) {
        //nonfairTryAcquire方法继承自Sync类
        return nonfairTryAcquire(acquires);
    }
    
    //Sync类的nonfairTryAcquire方法
    final boolean nonfairTryAcquire(int acquires) {
        //获取当前线程
        final Thread current = Thread.currentThread();
        //获取AQS中state当前值
        int c = getState();
        //如果state当前值等于0
        if (c == 0) {
            //使用CAS修改state值
            //如果此时state值等于0，也就是锁没有被其它线程持有，则修改成功
            if (compareAndSetState(0, acquires)) {
                //设置独占锁的当前所有者为当前线程
                setExclusiveOwnerThread(current);
                //获取到锁
                return true;
            }
        }
        //如果独占锁的当前所有者是当前线程，锁重入的情况
        else if (current == getExclusiveOwnerThread()) {
            //将state值加一
            int nextc = c + acquires;
            //判断新值是否溢出
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            //更新state值
            setState(nextc);
            //获取到锁
            return true;
        }
        //获取不到锁
        return false;
    }
    
    //上面分析的代码可以清晰的看到，当锁没有被线程持有时的获取过程，
    //但是，如果锁此时被其它线程持有，即执行tryAcquire方法返回false，
    //此时将需要先执行addWaiter方法，将当前线程封装成Node节点，并将这个Node节点插入到同步等待队列的尾部，
    //然后执行acquireQueued方法，阻塞当前线程，具体实现代码分析如下:
    
    //addWaiter方法继承自AQS
    //将当前线程封装成Node节点，并将这个Node节点插入到同步等待队列的尾部
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

    //acquireQueued方法继承自AQS
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取当前节点的前驱节点
                final Node p = node.predecessor();
                //如果当前节点的前驱节点是头结点，并且可以获取到锁，跳出循环并返回false
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
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

    //parkAndCheckInterrupt方法继承自AQS，用于阻塞当前线程
    private final boolean parkAndCheckInterrupt() {
        //阻塞当前线程，当前线程执行到这里即被挂起，等待被唤醒
        //当当前节点的线程被唤醒的时候，会继续尝试获取锁
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
当调用unlock()方法释放持有的锁时，具体实现代码如下。
```
    //ReentrantLock类的unlock方法
    public void unlock() {
        //此时的sync是NonfairSync的实例对象
        //在NonfairSync中没有重写release方法，release方法继承自AQS，所以执行AQS的release方法
        sync.release(1);
    }
    
    //AQS的release方法
    public final boolean release(int arg) {
        //尝试释放持有的锁
        //如果释放成功，则从同步等待队列的头结点开始唤醒等待线程
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //唤醒同步等待队列中的阻塞线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    //尝试释放持有的锁
    protected final boolean tryRelease(int releases) {
        //将state值减去1
        int c = getState() - releases;
        //如果当前线程不是持有锁的线程，抛异常
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //如果state的值等于零，则表明此锁已经完全被释放
        //如果state的值不等于零，则表明线程持有的锁(可重入锁)还没有完全被释放
        if (c == 0) {
            //free=true表示锁以被完全释放
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    //唤醒阻塞线程
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        //如果后继节点为空或已被取消，则从尾部开始找到等待队列中第一个waitStatus<=0，即未被取消的节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //唤醒等待队列节点中的线程
            //之前执行到parkAndCheckInterrupt方法的线程继续执行，再次尝试获取锁
            LockSupport.unpark(s.thread);
    }
```

## 原理总结
A、B两个线程同时执行lock()方法获取锁，假设A先执行获取到锁，此时state值加1，如果线程A在继续执行的过程中又执行了lock()方法，线程A会直接获取锁，同时state值加1，state的值可以简单理解为线程A执行lock()方法的次数；当线程B执行lock()方法获取锁时，会将线程B封装成Node节点，并将其插入到同步等待队列的尾部，然后阻塞当前线程，等待被唤醒再次尝试获取锁；线程A每次执行unlock()方法都会将state值减1，直到state的值等于零则表示完全释放掉了线程A持有的锁，此时将从同步等待队列的头节点开始唤醒阻塞的线程，阻塞线程恢复执行，再次尝试获取锁。ReentrantLock非公平锁的实现使用了AQS的同步等待队列和state。
