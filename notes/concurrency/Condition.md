### *Condition*
---
- 提供与Object.wait()和Object.notify()相似的功能，其与Lock相关联。
- 借助Condition对象，可以让线程在合适的时间等待，在某一时刻得到通知，继续执行。
- *Condition*实现有AbstractQueuedSynchronizer#ConditionObject和AbstractQueuedLongSynchronizer#ConditionObject
```java
public interface Condition {
	让当前线程等待，释放获得的锁；等待其它线程通知或者中断跳出
	void await() throws InterruptedException;
	等待过程中不响应中断
	void awaitUninterruptibly();
	有时间限定地等待，超时之后不再等待
	long awaitNanos(long nanosTimeout) throws InterruptedException;
	boolean await(long time, TimeUnit unit) throws InterruptedException;
	boolean awaitUntil(Date deadline) throws InterruptedException;

	唤醒一个等待线程
	void signal();
	唤醒所有等待线程
	void signalAll();
}
```