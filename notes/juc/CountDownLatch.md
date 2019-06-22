### CountDownLatch
---
CountDownLatch通常用于控制线程等待，等待一组线程完成任务后再进行其它操作。
- await()方法使调用线程等待；
- 线程组线程完成任务后调用countDown方法释放锁。

#### 1.CountDownLatch源码
```java
public class CountDownLatch {
	/* CountDownLatch的同步器(继承AQS) */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
        // 构造方法传入计数器值
        Sync(int count) { setState(count); }
        int getCount() { return getState(); }
        // 尝试获取锁：查询计数器(锁状态)值，只有当计数器值减到0时返回1，表示此时能够得到锁
        protected int tryAcquireShared(int acquires) { //入参没有使用到
            return (getState() == 0) ? 1 : -1;
        }
		// 尝试释放锁：即减少计数器值，但只有当减到0时，才返回true，此时表示所有计数线程均释放了锁
        protected boolean tryReleaseShared(int releases) { //入参没有使用到
            for (;;) {
                int c = getState(); 
                if (c == 0) //当计数器值已经为0，直接返回false
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc)) //CAS更新 
                    return nextc == 0; 
            }
        }
    }
    /* 结构 */
    private final Sync sync; //同步器

    // 构造函数，需要传入计数器值，用于设置相互等待线程数
    public CountDownLatch(int count) { 
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    // 等待一组线程完成任务，等待过程中通过轮询查看线程组是已经否完成任务。
    // 当调用线程可以获得到锁时，说明其线程组已经都已经release，释放了锁；
    // 锁状态由初始计数器值减为了0
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1); 
    }
    public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }
    // 释放锁
    public void countDown() { sync.releaseShared(1); }
    public long getCount() { return sync.getCount(); }
    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```
#### 2.await方法
```java
// 等待一组线程完成任务，等待过程中通过轮询查看线程组是已经否完成任务。
// 当调用线程可以获得到锁时，说明其线程组已经都已经release，释放了锁；
// 锁状态由初始计数器值减为了0
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1); 
}
// In AQS
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 只有state==0时，tryAcquireShared(1)，才返回1；其余返回-1
    if (tryAcquireShared(arg) < 0) 
        doAcquireSharedInterruptibly(arg);
}
// 自旋过程中，符合挂起条件时挂起线程；被唤醒后尝试获取锁...
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) { //自旋
            final Node p = node.predecessor();
            if (p == head) { // 尝试获取锁
                int r = tryAcquireShared(arg); // 1 or -1
                if (r >= 0) {
                    setHeadAndPropagate(node, r);//设置head并传播
                    p.next = null; // help GC
                    failed = false;
                    return; //返回
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) && 
                parkAndCheckInterrupt()) //若线程可挂起时，挂起线程
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
#### 3.countDown方法 
```java
public void countDown() {
    sync.releaseShared(1); //释放锁
}
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //只有最后一个线程完成时调用此方法才会返回true
        doReleaseShared();
        return true;
    }
    return false;
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h); // 唤醒后继等待线程(同步队列)
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); //唤醒线程
}
```

#### 4.一个例子
```java
public class CountDownLatchTest {
    private static CountDownLatch countDownLatch = new CountDownLatch(10);
    private static class CountDownTask implements Runnable{
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + " done...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            countDownLatch.countDown();
        }
    }
    private static class AwaitDoneTask implements Runnable{
        @Override
        public void run() {
            try {
                countDownLatch.await();
                System.out.println(Thread.currentThread().getName() + "wait done...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(8);
        for (int i = 0; i < 12 ; i++) {
            executor.submit(new CountDownTask());
        }
        // start three other threads to run await().
        for (int i = 0; i < 3 ; i++) {
            executor.submit(new AwaitDoneTask());
        }
        countDownLatch.await(); // main thread run await().
        System.out.println("work is fully done!");
        executor.shutdown();
    }
}
```