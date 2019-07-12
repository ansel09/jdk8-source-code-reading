### *CyclicBarrier*
---
一种多线程同步机制，让多个线程在barrier身上等待其它线程到达。

相较于CountDownLatch：
- CyclicBarrier增加了重置功能，可以正常执行完一轮自动重置，也可调用reset()方法进行重置。
- CountdownLatch中，每个调用await的线程，会一直被阻塞，直到其它线程通过countDown方法将计数器减至0；而CyclicBarrier则是有parties - 1个线程调用await方法会阻塞，直到最后一个线程调用await方法唤醒所有阻塞的线程。
- CyclicBarrier构造时可以设置barrierAction，让最后到达屏障点的线程执行。

#### 1. 结构
```java
private static class Generation { boolean broken = false; }
private final ReentrantLock lock = new ReentrantLock();
private final Condition trip = lock.newCondition(); //用于线程间通信：等待和唤醒
private final int parties; //线程数
private final Runnable barrierCommand; //最后到达屏障点的线程执行的Runnble任务(可选)
private Generation generation = new Generation(); //因为复用，设置一个代数
private int count; //仍在等待获取锁的线程，仍然未到达屏障点的线程
```
#### 2. 主要方法
##### 2.1 构造方法
构造对象时，可以传入barrierAction，让最后到达屏障点的线程执行动作。
```java
public CyclicBarrier(int parties) { this(parties, null); }
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties; //线程数
    this.count = parties; //计数
    this.barrierCommand = barrierAction;
}
```
##### 2.2 await方法
await()方法返回后，后面代码继续运行，此时表示这一组线程中所有线程都到达了Barrier。
```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
// 一组线程等待所有线程到达公共屏障点的动作
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock; 
    lock.lock(); // 必须获得锁
    try {
        final Generation g = generation;
        if (g.broken) // 屏障点破坏，抛出相应异常
            throw new BrokenBarrierException();
        if (Thread.interrupted()) { //中断
            breakBarrier(); // 如果有中断，当前代失效
            throw new InterruptedException();
        }
        int index = --count; //计数器值更新，当前仍在等待获取锁的线程数(唤醒之前)
        if (index == 0) {  // tripped ：所有线程都到达屏障点，当前线程是最后一个到达的
            boolean ranAction = false; 
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run(); //执行barrierCommand
                ranAction = true; //标记barrierCommand执行成功
                nextGeneration(); //这一代已经结束，开启下一代：其中会唤醒所有其它线程
                return 0; //返回0
            } finally {
                if (!ranAction) 
                    breakBarrier(); //如果barrierCommand运行失败，屏障失效
            }
        }
        // 当前线程不是最后一个到达线程，阻塞直到都到达屏障点，中断或超时
        for (;;) {
            try { //线程等待
                if (!timed)
                	// 等待在Condition上，释放锁，线程等待被唤醒竞争锁，
                	// 再次获得到锁的时候，方法返回。
                    trip.await(); 
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos); // 线程等待，释放锁
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) { 
                	//屏障当前代且未被破坏，
                	//表明其他线程还没有执行过breakBarrier()，
                	//则调用breakBarrier，其中会唤醒线程，并抛出异常。
                    breakBarrier(); //失效 
                    throw ie;
                } else {
                	// 不是当前代，说明当前代已经结束失效，
                	// 有其他线程执行了breakBarrier方法，
                	// 此种情况，中断线程，后面会抛出异常。
                    Thread.currentThread().interrupt();
                }
            }
            // 运行到这里的时候，最后到达屏障点的线程已经通知了所有线程；
            // 在await()返回时线程已经再次获得了锁，
            // 重新判断当前状态(因为在其他线程执行过程中，可能会出现异常、中断导致屏障失效)
            if (g.broken)
                throw new BrokenBarrierException();
            if (g != generation) // 正常执行的情况下，当前代已经结束
                return index; // 这一代已经结束，返回当前等待线程数，表示了到达屏障点顺序，
                // 最终调用栈继续返回，即await()方法返回。也就是await()之后的代码可以继续
                // 运行。
            if (timed && nanos <= 0L) {
                breakBarrier(); // 超时失效
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```
##### 2.3 reset方法
调用线程必须已经获得CyclicBarrier内的锁，然后使当前代失效，并开启新一代。
```java
// 重置
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try { // 以下两个方法，是在获得锁的前提下得到执行
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
// 设置当前屏障点失效，并通知唤醒所有线程；
// 注意前提是调用线程已经获得了锁。
private void breakBarrier() {
    generation.broken = true; //标记broken
    count = parties; //重置计数
    trip.signalAll(); //唤醒其他在trip条件下等待的线程
}
// 开启下一代：唤醒所有等待线程，并更新下一代设置
private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // 重置并开启下一代
    count = parties;
    generation = new Generation();
}
```
##### 2.4 其他
```java
// 判断屏障是否被破坏
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
// 已经到达屏障点的在等待的线程数(已经获取锁执行)
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```