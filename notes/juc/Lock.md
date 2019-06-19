#### *Lock*接口
---
与借助synchronized关键字的内置锁不同，Lock不是Java语言层面的，它是一个接口。可以用Lock来实现非公平锁，这是内置锁所不支持的。Lock使用上相对灵活，是显示锁。
```java
public interface Lock {
	// 获取锁，获取到锁后此方法返回，获取失败时线程等待队列自旋或者挂起，此方法不响应中断
	void lock();
	// 可中断的获取锁，可响应中断
    void lockInterruptibly() throws InterruptedException;

	// 尝试非阻塞地获取锁，方法会立即返回是否获取了锁 
    boolean tryLock();
    // 限时获取锁：time内获取到锁返回true，中断或超时返回false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    // 释放锁
    void unlock();
    // 创建一个与当前锁关联的Condition
	Condition newCondition();
}
```
*Lock*接口在java.util.concurrent.locks包的实现类有：
- ReentrantReadWriteLock#ReadLock和WriteLock
- ReentrantLock
- ConcurrentHashMap#Segment(继承ReentrantLock)
- StampedLock#ReadLockView和WriteLockView
