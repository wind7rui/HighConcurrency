
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
public void methodC(){
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

  public void methodC();
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter       // 正常获取锁
         4: aload_1
         5: monitorexit        // 正常释放锁
         6: goto          14   // 跳到14行，正常退出
         9: astore_2
        10: aload_1
        11: monitorexit        // 非正常退出时释放锁
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
         5: monitorenter      // 正常获取锁
         6: aload_1
         7: monitorexit       // 正常释放锁
         8: goto          16  // 跳到16行，正常退出
        11: astore_2
        12: aload_1
        13: monitorexit       // 非正常退出时释放锁
        14: aload_2
        15: athrow
        16: return
```
synchronized直接修饰在methodA和methodB的方法声明上，是方法级的同步，称为同步方法。通过反编译后的代码可以看到，methodA和methodB的访问标志flags中被标识了ACC_SYNCHRONIZED，methodA和methodB被调用时，调用指令会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程将请求获取同步锁，成功获取锁，开始执行同步代码，同步代码正常执行完成后或非正常结束后，执行线程都将释放持有的锁，此时其它线程可以继续请求获取锁。

synchronized作用在methodC和methodD内部，是语句级的同步，称为同步语句。通过反编译后的代码可以看到，methodC和methodD内部实现使用了Java虚拟机指令集中的monitorenter指令和monitorexit指令来实现synchronized的语义。当执行到synchronized修饰的语句块时，通过monitorenter指令获取锁，成功获取锁之后，执行同步代码，同步代码正常执行完成后，通过monitorexit指令释放持有的锁。JVM为了保证同步代码执行非正常结束时也释放持有的锁，所以，在发生异常时，再次通过monitorexit指令释放持有的锁。
