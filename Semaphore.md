
## Semaphore是什么
Semaphore是J.U.C包下的许可控制类，维护了一个许可集，通常用于限制可以访问某些资源（物理或逻辑的）的线程数目，或对资源访问的许可控制。

## 常用方法
### acquire()
从许可集中请求获取一个许可，此时当前线程开始阻塞，直到获得一个可用许可，或者当前线程被中断。

acquire(int permits) 
从许可集中请求获取指定数目(permits)的许可，此时当前线程开始阻塞，直到获得指定数据(permits)可用许可，或者当前线程被中断。

release() 
释放一个许可，将其返回给许可集。

release(int permits) 
释放指定数据(permits)许可，将其返回给许可集。

tryAcquire() 


tryAcquire(int permits) 


tryAcquire(int permits, long timeout, TimeUnit unit) 


tryAcquire(long timeout, TimeUnit unit) 
