
## 概述
Condition接口位于java.util.concurrent.locks包下，实现类有
AbstractQueuedLongSynchronizer.ConditionObject和
AbstractQueuedSynchronizer.ConditionObject。Condition将Object监视器方法(wait、notify和 notifyAll)分解成截然不同的对象，以便通过将这些对象与任意Lock实现组合使用。其中，Lock替代了synchronized方法的使用及作用，Condition替代了Object监视器方法的使用及作用。Condition的await方法代替Object的wait；Condition的signal方法代替Object的notify方法；Condition的signalAll方法代替Object的notifyAll方法。Condition实例在使用时需要绑定到一个锁上，可以通过newCondition方法获取Condition实例。Condition实现可以提供不同于Object监视器方法的行为和语义，比如受保证的通知排序，或者在执行通知时不需要保持一个锁。

## 样例代码
下面的代码演示了Condition简单使用的样例。
```
public class ConditionDemo {
    @Test    
    public void test() {
        final ReentrantLock reentrantLock = new ReentrantLock();
        final Condition condition = reentrantLock.newCondition();        
        new Thread(new Runnable() {
            @Override            
            public void run() {                
                try {
                    reentrantLock.lock();
                    System.out.println(Thread.currentThread().getName() + "在等待被唤醒");
                    condition.await();
                    System.out.println(Thread.currentThread().getName() + "恢复执行了");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    reentrantLock.unlock();
                }
            }
        }, "thread1").start();        
        new Thread(new Runnable() {
            @Override            
            public void run() {                
                try {
                    reentrantLock.lock();
                    System.out.println(Thread.currentThread().getName() + "抢到了锁");
                    condition.signal();
                    System.out.println(Thread.currentThread().getName() + "唤醒其它等待的线程");
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    reentrantLock.unlock();
                }
            }
        }, "thread2").start();
    }
}
```
输出结果如下所示：
```
thread1在等待被唤醒
thread2抢到了锁
thread2唤醒其它等待的线程
thread1恢复执行了
```

## 创建Condition实例
通过Lock接口实现类的newCondition方法获取Condition实例，例如如下代码：
```
ReentrantLock reentrantLock = new ReentrantLock();
Condition condition = reentrantLock.newCondition();
```

## 常用方法
### await()
调用await方法后，当前线程在接收到唤醒信号之前或被中断之前一直处于等待休眠状态。调用此方法时，当前线程保持了与此Condition有关联的锁，调用此方法后，当前线程释放持有的锁。此方法可以返回当前线程之前，都必须重新获取与此条件有关的锁，在线程返回时，可以保证它保持此锁。

### await(long time,TimeUnit unit)
调用此方法后，会造成当前线程在接收到唤醒信号之前、被中断之前或到达指定等待时间之前一直处于等待状态。调用此方法时，当前线程保持了与此Condition有关联的锁，调用此方法后，当前线程释放持有的锁。time参数为最长等待时间；unit参数为time的时间单位。如果在从此方法返回前检测到等待时间超时，则返回 false，否则返回true。此方法可以返回当前线程之前，都必须重新获取与此条件有关的锁，在线程返回时，可以保证它保持此锁。

### awaitNanos(long nanosTimeout)
该方法等效于await(long time,TimeUnit unit)方法，只是等待的时间是
nanosTimeout指定的以毫微秒数为单位的等待时间。该方法返回值是所剩毫微秒数的一个估计值，如果超时，则返回一个小于等于0的值。可以根据该返回值来确定是否要再次等待，以及再次等待的时间。

### awaitUninterruptibly()
调用此方法后，会造成当前线程在接收到唤醒信号之前一直处于等待状态。如果在进入此方法时设置了当前线程的中断状态，或者在等待时，线程被中断，那么在接收到唤醒信号之前，它将继续等待。当最终从此方法返回时，仍然将设置其中断状态。调用此方法时，当前线程保持了与此Condition有关联的锁，调用此方法后，当前线程释放持有的锁。此方法可以返回当前线程之前，都必须重新获取与此条件有关的锁，在线程返回时，可以保证它保持此锁。

### awaitUntil(Date deadline)
调用此方法后，会造成当前线程在接收到唤醒信号之前、被中断之前或到达指定最后期限之前一直处于等待休眠状态。调用此方法时，当前线程保持了与此Condition有关联的锁，调用此方法后，当前线程释放持有的锁。此方法可以返回当前线程之前，都必须重新获取与此条件有关的锁，在线程返回时，可以保证它保持此锁。

### signal()
唤醒一个等待线程，如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从await返回之前，该线程必须重新获取锁。

### signalAll()
唤醒所有等待线程，如果所有的线程都在等待此条件，则唤醒所有线程。 在从await返回之前，每个线程必须重新获取锁。

## 底层实现原理
这里以AQS内部的ConditionObject实现为例，分析底层实现原理，这部分内容已在[高并发编程-CyclicBarrier深入解析](https://github.com/wind7rui/HighConcurrency/blob/master/CyclicBarrier.md)后半部分深入分析过，这里不再做具体分析。


