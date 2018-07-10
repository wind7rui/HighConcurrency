
CAS（Compare and swap）直译过来就是比较和替换，是一种通过硬件实现并发安全的常用技术，底层通过利用CPU的CAS指令对缓存加锁或总线加锁的方式来实现多处理器之间的原子操作。仔细观察J.U.C包中类的实现代码，会发现这些类中大量使用到了CAS，所以CAS是Java并发包的实现基础。它的实现过程是，有3个操作数，内存值V，旧的预期值E，要修改的新值U，当且仅当预期值E和内存值V相同时，才将内存值V修改为U，否则什么都不做。

## CAS底层实现原理
下面以AtomicInteger为入口来看一下CAS的底层实现原理。AtomicInteger可以用原子方式更新其int类型的属性值value。其中，incrementAndGet方法以原子方式将当前值加1，并返回最新值，具体代码如下。
```
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }

    public final int get() {
        return value;
    }

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```
上面的代码中有一个旧的期望值expect和将要更新的新值update。实现的关键代码就在compareAndSet方法中，可以看到这里调用了Unsafe类的compareAndSwapInt方法，进入这个方法发现使用了native修饰，也就是这里将会通过JNI调用非Java实现的代码。
```
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```
继续深入代码进行分析，就需要进入OpenJDK的源码查看c++的代码，代码跟踪顺序是：unsafe.cpp、atomic.cpp，接下来会根据操作系统和处理器的不同来选择对应的调用代码，这里以Windows和x86处理器为例进入atomic_window_x86.inline.hpp，重要代码片段如下：
```
inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {
    int mp = os::isMP();  // 判断处理器的类型是否是多处理器
    _asm {
        mov edx, dest
        mov ecx, exchange_value
        mov eax, compare_value
        LOCK_IF_MP(mp)
        cmpxchg dword ptr [edx], ecx
    }
}
```
从上面代码可以看到，如果是多处理器，将会为cmpxchg指令添加lock前缀，否则不添加(单处理器自身会维护执行顺序)。对于lock前缀，下面是intel手册的说明：
- 确保对内存读改写操作的原子执行。
在Pentium及之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其它处理器暂时无法通过总线访问内存，很显然，这个开销很大。在新的处理器中，Intel使用缓存锁定来保证指令执行的原子性。缓存锁定将大大降低lock前缀指令的执行开销。
- 禁止该指令，与前面和后面的读写指令重排序。
- 把写缓冲区的所有数据刷新到内存中。

也就是说，如果是多处理器，通过带lock前缀的cmpxchg指令对缓存加锁或总线加锁的方式来实现多处理器之间的原子操作；如果是单处理器，通过cmpxchg指令完成原子操作。

## ABA问题
CAS是当且仅当旧的预期值E和内存值V相同时，才将内存值V修改为U，也就是如果内存值V没有发生变化则更新，但是有可能发生内存值原来是A，中间被改成B，后来又被改成A，此时再使用CAS进行检查时发现没有变化，但是实际上发生了变化，这就是ABA问题。

Java并发包下的AtomicStampedReference可以解决ABA问题，内部实现上添加了一个类似于版本号作用的stamp属性，它是被自动更新的。实现上首先检查当前引用是否等于预期引用、当前stamp是否等于预期stamp，如果全部相等，则以原子方式将该引用和该stamp的值设置为给定的更新值。
