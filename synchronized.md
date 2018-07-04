
## synchronized概述
对于单一JVM来说，synchronized可以保证在并发情况下，同一时刻只有一个线程执行某个方法或某段代码。synchronized可用于修饰普通方法、静态方法和代码块，都可以实现对同步代码的并发安全控制。

synchronized修饰普通方法时，同步代码执行前，需要获取当前实例对象的锁(对象锁)。
```
public synchronized void  methodA(){
    //省略同步代码
}
```

synchronized修饰静态方法时，同步代码执行前，需要获取当前类的锁(类锁)。
```
public static synchronized void  methodB(){
    //省略同步代码
}
```

第一种修饰代码块，对实例对象加锁，同步代码执行前，需要获取当前实例对象的锁(对象锁)。
```
public void methodaC(){
	//this可以替换成其它实例对象
    synchronized (this){
    	//省略同步代码
    }
}
```

第二种修饰代码块，对Class加锁，同步代码执行前，需要获取当前类的锁(类锁)。
```
public void methodD() {
    synchronized (Xxx.class) {
    	//省略同步代码
    }
}
```

## synchronized底层实现
使用javap -c -v命令对class文件进行反编译，可以看到上面四个方法的底层实现(这里省略不必要的信息)。
