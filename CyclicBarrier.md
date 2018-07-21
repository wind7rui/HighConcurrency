## 要点解说
CyclicBarrier是一个同步辅助类，它允许一组线程互相等待，直到所有线程都到达某个公共屏障点(也可以叫同步点)，即相互等待的线程都完成调用await方法，所有被屏障拦截的线程才会继续运行await方法后面的程序。在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时CyclicBarrier很有用。因为该屏障点在释放等待线程后可以重用，所以称它为循环的屏障点。CyclicBarrier支持一个可选的Runnable命令，在一组线程中的最后一个线程到达屏障点之后（但在释放所有线程之前），该命令只在所有线程到达屏障点之后运行一次，并且该命令由最后一个进入屏障点的线程执行。

## 实例演示
CyclicBarrier简单使用样例。
```
public class CyclicBarrierDemo {

    @Test
    public void test() {
        final CyclicBarrier barrier = new CyclicBarrier(2, myThread);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName());
                    barrier.await();
                    System.out.println(Thread.currentThread().getName());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, "thread1").start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName());
                    barrier.await();
                    System.out.println(Thread.currentThread().getName());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, "thread2").start();
    }

    Thread myThread = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("myThread");
        }
    }, "thread3");
}
```
结果输出：
```
thread1
thread2
myThread
thread2
thread1
```

## 方法解析
1.CyclicBarrier(int parties, Runnable barrierAction)
创建一个CyclicBarrier实例，parties指定参与相互等待的线程数，barrierAction指定当所有线程到达屏障点之后，首先执行的操作，该操作由最后一个进入屏障点的线程执行。

2.CyclicBarrier(int parties)
创建一个CyclicBarrier实例，parties指定参与相互等待的线程数。

3.getParties()
返回参与相互等待的线程数。

4.await()
该方法被调用时表示当前线程已经到达屏障点，当前线程阻塞进入休眠状态，直到所有线程都到达屏障点，当前线程才会被唤醒。

5.await(long timeout, TimeUnit unit)
该方法被调用时表示当前线程已经到达屏障点，当前线程阻塞进入休眠状态，在timeout指定的超时时间内，等待其他参与线程到达屏障点；如果超出指定的等待时间，则抛出TimeoutException异常，如果该时间小于等于零，则此方法根本不会等待。

6.isBroken()
判断此屏障是否处于中断状态。如果因为构造或最后一次重置而导致中断或超时，从而使一个或多个参与者摆脱此屏障点，或者因为异常而导致某个屏障操作失败，则返回true；否则返回false。

7.reset()
将屏障重置为其初始状态。

8.getNumberWaiting()
返回当前在屏障处等待的参与者数目，此方法主要用于调试和断言。

## 源码解析
CyclicBarrier(int parties, Runnable barrierAction)和await()方法是CyclicBarrier的核心，本篇重点分析这两个方法的背后实现原理。
首先，看一下CyclicBarrier内声明的一些属性信息：
```
//用于保护屏障入口的锁
private final ReentrantLock lock = new ReentrantLock();
//线程等待条件
private final Condition trip = lock.newCondition();
//记录参与等待的线程数
private final int parties;
//当所有线程到达屏障点之后，首先执行的命令
private final Runnable barrierCommand;
private Generation generation = new Generation();
//实际中仍在等待的线程数，每当有一个线程到达屏障点，count值就会减一；当一次新的运算开始后，count的值被重置为parties
private int count;
```
其中，Generation是CyclicBarrier的一个静态内部类，它只有一个boolean类型的属性，具体代码如下：
```
    private static class Generation {
        boolean broken = false;
    }
```
当使用构造方法创建CyclicBarrier实例的时候，就是给上面这些属性赋值，
```
   //创建一个CyclicBarrier实例，parties指定参与相互等待的线程数，
   //barrierAction指定当所有线程到达屏障点之后，首先执行的操作，该操作由最后一个进入屏障点的线程执行。
   public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    //创建一个CyclicBarrier实例，parties指定参与相互等待的线程数
    public CyclicBarrier(int parties) {
        this(parties, null);
    }
```
当调用await()方法时，当前线程已经到达屏障点，当前线程阻塞进入休眠状态，
```
    //该方法被调用时表示当前线程已经到达屏障点，当前线程阻塞进入休眠状态
    //直到所有线程都到达屏障点，当前线程才会被唤醒
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen;
        }
    }

    //该方法被调用时表示当前线程已经到达屏障点，当前线程阻塞进入休眠状态
    //在timeout指定的超时时间内，等待其他参与线程到达屏障点
    //如果超出指定的等待时间，则抛出TimeoutException异常，如果该时间小于等于零，则此方法根本不会等待
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        //使用独占资源锁控制多线程并发进入这段代码
        final ReentrantLock lock = this.lock;
        //独占锁控制线程并发访问
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();
            //如果线程中断，则唤醒所有等待线程
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
           //每调用一次await()方法，计数器就减一
           int index = --count;
           //当计数器值等于0的时
           if (index == 0) {  // tripped
               boolean ranAction = false;
               try {
                   final Runnable command = barrierCommand;
                   //如果在创建CyclicBarrier实例时设置了barrierAction，则先执行barrierAction
                   if (command != null)
                       command.run();
                   ranAction = true;
                   //当所有参与的线程都到达屏障点，为唤醒所有处于休眠状态的线程做准备工作
                   //需要注意的是，唤醒所有阻塞线程不是在这里
                   nextGeneration();
                   return 0;
               } finally {
                   if (!ranAction)
                       breakBarrier();
               }
           }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        //让当前执行的线程阻塞，处于休眠状态
                        trip.await();
                    else if (nanos > 0L)
                        //让当前执行的线程阻塞，在超时时间内处于休眠状态
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            //释放独占锁
            lock.unlock();
        }
    }
    
    private void nextGeneration() {
        //为唤醒所有处于休眠状态的线程做准备工作
        trip.signalAll();
        //重置count值为parties
        count = parties;
        //重置中断状态为false
        generation = new Generation();
    }

    private void breakBarrier() {
        //重置中断状态为true
        generation.broken = true;
        //重置count值为parties
        count = parties;
        //为唤醒所有处于休眠状态的线程做准备工作
        trip.signalAll();
    }
```
到这里CyclicBarrier的实现原理基本已经都清楚了，下面来深入源码分析一下线程阻塞代码trip.await()和线程唤醒trip.signalAll()的实现。
```
        //await()是AQS内部类ConditionObject中的方法
        public final void await() throws InterruptedException {
            //如果线程中断抛异常
            if (Thread.interrupted())
                throw new InterruptedException();
            //新建Node节点，并将新节点加入到Condition等待队列中
            //Condition等待队列是AQS内部类ConditionObject实现的，ConditionObject有两个属性，分别是firstWaiter和lastWaiter，都是Node类型
            //firstWaiter和lastWaiter分别用于代表Condition等待队列的头结点和尾节点
            Node node = addConditionWaiter();
            //释放独占锁，让其它线程可以获取到dowait()方法中的独占锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //检测此节点是否在资源等待队列(AQS同步队列)中，
            //如果不在，说明此线程还没有竞争资源锁的权利，此线程继续阻塞，直到检测到此节点在资源等待队列上(AQS同步队列)中
            //这里出现了两个等待队列，分别是Condition等待队列和AQS资源锁等待队列(或者说是同步队列)
            //Condition等待队列是等待被唤醒的线程队列，AQS资源锁等待队列是等待获取资源锁的队列
            while (!isOnSyncQueue(node)) {
                //阻塞当前线程，当前线程进入休眠状态，可以看到这里使用LockSupport.park阻塞当前线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        //addConditionWaiter()是AQS内部类ConditionObject中的方法
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 将condition等待队列中，节点状态不是CONDITION的节点，从condition等待队列中移除
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            //以下操作是用此线程构造一个节点，并将之加入到condition等待队列尾部
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
        
        //signalAll是AQS内部类ConditionObject中的方法
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            //Condition等待队列的头结点
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
        
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                //将Condition等待队列中的Node节点按之前顺序都转移到了AQS同步队列中
                transferForSignal(first);
                first = next;
            } while (first != null);
        }

        final boolean transferForSignal(Node node) {
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
               return false;
            //这里将Condition等待队列中的Node节点插入到AQS同步队列的尾部
            Node p = enq(node);
            int ws = p.waitStatus;
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
               LockSupport.unpark(node.thread);
            return true;
       }

       //ReentrantLock#unlock()方法
       public void unlock() {
           //Sync是ReentrantLock的内部类，继承自AbstractQueuedSynchronizer，它是ReentrantLock中公平锁和非公平锁的基础实现
           sync.release(1);
       }

       public final boolean release(int arg) {
           //释放锁
           if (tryRelease(arg)) {
               //AQS同步队列头结点
               Node h = head;
               if (h != null && h.waitStatus != 0)
                   //唤醒节点中的线程
                   unparkSuccessor(h);
               return true;
           }
           return false;
       }

       private void unparkSuccessor(Node node) {
           int ws = node.waitStatus;
           if (ws < 0)
               compareAndSetWaitStatus(node, ws, 0);
           Node s = node.next;
           if (s == null || s.waitStatus > 0) {
               s = null;
               for (Node t = tail; t != null && t != node; t = t.prev)
                   if (t.waitStatus <= 0)
                       s = t;
           }
           if (s != null)
               //唤醒阻塞线程
               LockSupport.unpark(s.thread);
    }
       
```
## 原理总结
用上面的示例总结一下CyclicBarrier的await方法实现，假设线程thread1和线程thread2都执行到CyclicBarrier的await()，都进入dowait(boolean timed, long nanos)，thread1先获取到独占锁，执行到--count的时，index等于1，所以进入下面的for循环，接着执行trip.await()，进入await()方法，执行Node node = addConditionWaiter()将当前线程构造成Node节点并加入到Condition等待队列中，然后释放获取到的独占锁，当前线程进入阻塞状态；此时，线程thread2可以获取独占锁，继续执行--count，index等于0，所以先执行command.run()，输出myThread，然后执行nextGeneration()，nextGeneration()中trip.signalAll()只是将Condition等待队列中的Node节点按之前顺序都转移到了AQS同步队列中，这里也就是将thread1对应的Node节点转移到了AQS同步队列中，thread2执行完nextGeneration()，返回return 0之前，细看代码还需要执行lock.unlock()，这里会执行到ReentrantLock的unlock()方法，最终执行到AQS的unparkSuccessor(Node node)方法，从AQS同步队列中的头结点开始释放节点，唤醒节点对应的线程，即thread1恢复执行。

如果有三个线程thread1、thread2和thread3，假设线程执行顺序是thread1、thread2、thread3，那么thread1、thread2对应的Node节点会被加入到Condition等待队列中，当thread3执行的时候，会将thread1、thread2对应的Node节点按thread1、thread2顺序转移到AQS同步队列中，thread3执行lock.unlock()的时候，会先唤醒thread1，thread1恢复继续执行，thread1执行到lock.unlock()的时候会唤醒thread2恢复执行。

## 实战经验
一个excel有多个sheet，每个sheet记录用户的每日交易流水，如果要计算这个用户当月的日平均消费情况，可以使用多线程先分别计算每日的消费情况，然后再做汇总计算平均值。

## 面试考点
CyclicBarrier当所有线程都到达屏障点后，等待线程的执行顺序是什么样的？

CyclicBarrier的await方法是使用ReentrantLock和Condition控制实现的，使用的Condition实现类是ConditionObject，它里面有一个等待队列和await方法，这个await方法会向队列中加入元素。当调用CyclicBarrier的await方法会间接调用ConditionObject的await方法，当屏障关闭后首先执行指定的barrierAction，然后依次执行等待队列中的任务，有先后顺序。
