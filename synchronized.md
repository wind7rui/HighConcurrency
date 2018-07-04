
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
```
 public synchronized void methodA();
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 8: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
               0       1     0  this   LSyncTest;

  public static synchronized void methodB();
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 12: 0

  public void methodaC();
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_1
         5: monitorexit
         6: goto          14
         9: astore_2
        10: aload_1
        11: monitorexit
        12: aload_2
        13: athrow
        14: return

  public void methodD();
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc_w         #2  // class Xxx
         3: dup
         4: astore_1
         5: monitorenter
         6: aload_1
         7: monitorexit
         8: goto          16
        11: astore_2
        12: aload_1
        13: monitorexit
        14: aload_2
        15: athrow
        16: return
```
