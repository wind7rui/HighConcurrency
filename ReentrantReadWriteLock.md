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

### ReadLock类


### WriteLock类

## 源码解析

