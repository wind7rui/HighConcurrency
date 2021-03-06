## 底层实现原理
ThreadLocal的实现原理是每一个Thread维护一个ThreadLocalMap映射表，映射表的key是ThreadLocal实例，并且使用的是ThreadLocal的弱引用 ，value是具体需要存储的Object。下面用一张图展示这些对象之间的引用关系，实心箭头表示强引用，空心箭头表示弱引用。

![ThreadLocal底层实现原理](https://github.com/wind7rui/HighConcurrency/blob/master/ThreadLocal.jpg)

## 内存泄漏问题
从上图可以看出，如果ThreadLocal没有外部强引用，当发生垃圾回收时，这个ThreadLocal一定会被回收(弱引用的特点是不管当前内存空间足够与否，GC时都会被回收)，这样就会导致ThreadLocalMap中出现key为null的Entry，外部将不能获取这些key为null的Entry的value，并且如果当前线程一直存活，那么就会存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value，导致value对应的Object一直无法被回收，产生内存泄露。

查看源码会发现，ThreadLocal的get、set和remove方法都实现了对所有key为null的value的清除，但仍可能会发生内存泄露，因为可能使用了ThreadLocal的get或set方法后发生GC，此后不调用get、set或remove方法，为null的value就不会被清除。

解决办法是每次使用完ThreadLocal都调用它的remove()方法清除数据，或者按照JDK建议将ThreadLocal变量定义成private static，这样就一直存在ThreadLocal的强引用，也就能保证任何时候都能通过ThreadLocal的弱引用访问到Entry的value值，进而清除掉。
