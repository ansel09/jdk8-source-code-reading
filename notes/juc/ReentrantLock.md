### *ReentrantLock*
---

ReentrantLock是可重入锁的实现，提供了与synchronized相似的功能，不同的是需要显示的去获取与释放锁。

#### 1. 内部同步器Sync与其子类

 ReentrantLock的抽象静态内部类Sync继承了AQS
##### 1.1 Sync
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;
    // 获取锁，抽象方法，由子类FairSync和UnfairSync实现
	abstract void lock();

    // 实现非公平的tryLock，尝试获取锁。AQS模板方法tryAcquire()由Sync子类实现
    // 公平锁非公平锁都需要此方法，因此提到公共基类中进行定义
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 锁没有被占用
        if (c == 0) {
        	// 尝试获取同步锁
            if (compareAndSetState(0, acquires)) { 
            	// AbstractOwnableSynchronizer#setExclusiveOwnerThread()
            	// 设置当前线程为获取到锁的线程
                setExclusiveOwnerThread(current);
                // 获取成功返回
                return true;
            }
        }
        // AbstractOwnableSynchronizer#getExclusiveOwnerThread()获取到同步锁资源的线程
        else if (current == getExclusiveOwnerThread()) { //当前线程已经获取到锁，重入
            int nextc = c + acquires; // 累加同步锁状态值
            if (nextc < 0) // 状态值overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc); // 设置状态值
            //锁重入，返回
            return true;
        }
        // 获取锁失败返回
        return false;
    }

    // tryRelease()为AQS类中定义的模板方法，释放锁
    // 因为锁的可重入，线程多次获取锁，需要释放至同步状态为0
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases; //考虑锁重入的情况
        // 检查，显然在释放锁时必须是获得锁的线程
        if (Thread.currentThread() != getExclusiveOwnerThread()) 
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) { // 释放锁成功，同步状态为0
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    // 线程是否独占锁资源
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    // 创建锁关联的Condition 
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // 以下方法供外部类使用：
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }
    final boolean isLocked() {
        return getState() != 0; //同步状态为0时锁没有被占用
    }

	// 从输入流中构建实例 
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

##### 1.2 NonfairSync
```java
// NonfairSync非公平锁
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
	// 获取锁
    final void lock() {
    	// 尝试快速获取：直接CAS修改锁同步状态
        if (compareAndSetState(0, 1))
        	// CAS修改状态成功，标记当前线程获取到同步资源 
            setExclusiveOwnerThread(Thread.currentThread());

    	// 快速获取失败，再次获取；成功返回，失败进入同步队列
        else
            acquire(1); // 独占模式，传入状态值为1
    }

	// AQS中定义的模板方法
    protected final boolean tryAcquire(int acquires) {
    	// 调用父类中的方法
        return nonfairTryAcquire(acquires);
    }
}
```
##### 1.3 FairSync
```java
// Sync的公平锁实现
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    // 阻塞获取锁，直到获取锁后，方法才返回
    final void lock() {
    	// acquire()方法定义在基类AQS中，根据实际可能会调用到以下方法
    	// tryAcquire(); addWaiter(); acquireQueued()
        acquire(1);
    }
    // 重写AQS的模板方法tryAcquire
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) { // 锁没有被占用
        	// AQS#hasQueuedPredecessors()，
        	// 判断一下队列中有没有更早的处于等待状态的结点
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) { 
            	// 获取成功，标识当前线程占用锁
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) { //锁重入
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 获取失败快速返回
        return false;
    }
}
```

#### 2. ReentrantLock的构造函数

ReentrantLock默认是非公平的，其有两个构造函数：
```java
private final Sync sync; 
// 默认为非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
// 选择锁是公平还是非公平的
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```