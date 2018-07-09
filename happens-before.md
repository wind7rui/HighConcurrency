
happens-before是Java内存模型中定义的两个操作之间的偏序关系，即如果操作A在操作B之前先发生，那么操作A产生的操作结果，操作B可以观察到，或者说操作A的结果影响到操作B。笔者认为Java内存模型中的这种与生俱来的原则实现了可见性和顺序性。

happens-before无需任何同步器的协助，只要两个操作之间的关系符合以下列出的这些规则，或者可以由以下这些规则推导出，那么就可以保证它们的顺序性，否则Java虚拟机可以进行任意重排序。

## 程序次序规则
在一个线程内，按照控制流顺序，书写在前面的操作Happens-Before书写在后面的操作
## 管程锁定规则
一个unlock操作Happens-Before后面对同一个锁的lock操作。
## volatile变量规则
对一个volatile变量的写入操作Happens-Before后面对这个变量的读操作。
## 线程启动规则
Thread对象的start()方法Happens-Before此线程的每一个动作。
## 线程终止规则
线程中的所有操作都Happens-Before对此线程的终止检测。
## 线程中断规则
对线程interrupt()方法的调用Happens-Before被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupt()方法检测到是否有中断发生。
## 对象终结规则
一个对象的初始化完成（构造函数执行结束）Happens-Before它的finalize()方法的开始。
## 传递性
如果操作A先行发生于操作B，操作B先行发生于操作C，那么就可以得出操作A先行发生于操作C。
