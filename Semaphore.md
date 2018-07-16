
## Semaphore是什么
Semaphore是J.U.C包下的许可控制类，维护了一个许可集，通常用于限制可以访问某些资源（物理或逻辑的）的线程数目，或对资源访问的许可控制。

## 常用方法
acquire()：从许可集中请求获取一个许可，此时当前线程开始阻塞，直到获得一个可用许可，或者当前线程被中断。

acquire(int permits)：从许可集中请求获取指定个数(permits)的许可，此时当前线程开始阻塞，直到获得指定数据(permits)可用许可，或者当前线程被中断。

release()：释放一个许可，将其返回给许可集。

release(int permits)：释放指定个数(permits)许可，将其返回给许可集。

tryAcquire()：尝试获取一个可用许可，如果此时有一个可用的许可，则立即返回true，同时许可集中许可个数减一；如果此时许可集中无可用许可，则立即返回false。

tryAcquire(int permits)：尝试获取指定个数(permits)可用许可，如果此时有指定个数(permits)可用的许可，则立即返回true，同时许可集中许可个数减指定个数(permits)；如果此时许可集中许可个数不足指定个数(permits)，则立即返回false。

tryAcquire(long timeout, TimeUnit unit)：在给定的等待时间内，尝试获取一个可用许可，如果此时有一个可用的许可，则立即返回true，同时许可集中许可个数减一；如果此时许可集中无可用许可，当前线程阻塞，直至其它某些线程调用此Semaphore的release()方法并且当前线程是下一个被分配许可的线程，或者其它某些线程中断当前线程，或者已超出指定的等待时间。

tryAcquire(int permits, long timeout, TimeUnit unit)：在给定的等待时间内，尝试获取指定个数(permits)可用许可，如果此时有指定个数(permits)可用的许可，则立即返回true，同时许可集中许可个数减指定个数(permits)；如果此时许可集中许可个数不足指定个数(permits)，当前线程阻塞，直至其它某些线程调用此Semaphore的release()方法并且当前线程是下一个被分配许可的线程并且许可个数满足指定个数，或者其它某些线程中断当前线程，或者已超出指定的等待时间。

## 实现原理
Semaphore内部原理是通过AQS实现的。Semaphore中定义了Sync抽象类，而Sync又继承了AbstractQueuedSynchronizer，Semaphore中对许可的获取与释放，是使用CAS通过对AQS中state的操作实现的。
![Semaphore](https://github.com/wind7rui/HighConcurrency/blob/master/Semaphore.png)

Semaphore对许可的分配有两种策略，公平策略和非公平策略，没有明确指明时，默认为非公平策略。

公平策略：根据方法调用顺序（即先进先出；FIFO）来选择线程、获得许可。
非公平策略：不对线程获取许可的顺序做任何保证。

Semaphore提供了两个构造方法用于构建实例对象。
```
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
    
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

下面分析非公平策略的Semaphore实现。首先是acquire()方法的实现代码。
```
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```
acquireSharedInterruptibly方法的实现在AQS中。
```
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        // 尝试获取指定个数许可
        // 如果许可个数不足，则执行doAcquireSharedInterruptibly
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```
可以看到，如果当前线程被中断，则直接抛出中断异常，否则继续执行tryAcquireShared方法，tryAcquireShared方法对公平策略和非公平策略在Semaphore中有不同的实现，这里分析非公平策略的实现，进入Semaphore的静态内部类NonfairSync中查看tryAcquireShared具体实现。
```
    static final class NonfairSync extends Sync {
        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
```
nonfairTryAcquireShared方法继承自Sync。
```
    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```
可以看到，Semaphore中许可的分配是通过AQS中的state实现的。创建Semaphore对象时，初始化AQS的state值；当向Semaphore对象请求获取许可时，会获取state当前值，然后用当前值减去要获取的许可个数，得到许可剩余个数，如果剩余个数不足(小于0)或者剩余个数充足并且通过CAS成功修改state值，则直接返回许可剩余个数，否则一直做轮训获取操作。

回到上面的acquireSharedInterruptibly方法，如果此时方法中tryAcquireShared执行结果是大于等于0，则获取许可成功，否则执行doAcquireSharedInterruptibly方法，这个方法的实现在AQS中。
```
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
把当前线程封装成Node节点，并加入到等待队列的尾部，通过循环再次尝试获取许可，如果不能获取则当前线程阻塞，否则恢复当前线程并返回。

无参数的tryAcquire方法和有参数的tryAcquire方法，在具体实现上和acquire方法类似，这里不再做具体分析。下面分析release方法。
```
    public void release() {
        sync.releaseShared(1);
    }
```
releaseShared方法的具体实现在AQS中。
```
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
tryReleaseShared方法在Sync类中进行了重写。
```
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
       }
    }
```
如果要归还的许可个数和当前剩下的许可个数的总和超限，则抛出Error；否则通过CAS修改state，成功则返回true，失败返回false。返回到上面的releaseShared方法，如果tryReleaseShared方法执行返回false，则直接返回false，归还许可失败；如果tryReleaseShared方法执行返回true，则可以继续进行归还许可操作，执行doReleaseShared方法。
```
	private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
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
doReleaseShared方法中，从等待队列的头结点开始，恢复符合条件的阻塞线程，使其恢复继续执行。
