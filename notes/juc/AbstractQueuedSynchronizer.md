### *AQS*
---
*AbstractQueuedSynchronizer, AQS*是J.U.C中许多同步工具的基类。它是一个基于FIFO队列，可用来构建锁或其它同步工具的基础框架。

AQS使用了模板方法模式，即AQS类实现了有关同步整体流程的骨架，如具体线程等待队列的维护：获取失败入队、唤醒出队等；而细节部分 - 共享资源state的获取与释放方式由不同的子类重写相关模板方法完成。

#### 1. *AQS*主要概念
##### 1.1 AQS定义的资源共享方式
+ 独占(Exlusive): 只有一个线程能够获得资源得到执行
+ 共享(Share)：多个线程可以获得资源而同时执行

##### 1.2 *AQS* 同步状态
*AQS*使用一个 *int* 类型 *volatile* 成员变量`state`来表示同步状态。并提供了读写方法以及*CAS*修改的方法：
- `protected final int getState()`
- `protected final void setState()`
- `protected final boolean compareAndSetState(int expect, int update)`

##### 1.3 模板方法
- `tryAcquire(int)` 独占：尝试获取资源，成功返回true，失败返回false
- `tryRelease(int)` 独占：尝试释放资源，成功返回true，失败返回false
- `tryAcquireShared(int)` 共享：尝试获取资源，返回负数为失败；0为成功但无剩余可用资源；正数为成功且有剩余资源。
- `tryReleaseShared(int)` 共享：尝试释放资源，成功返回true，失败返回false
- `isHeldExclusively()` 判断该线程是否正在独占资源，只在ConditionObject的方法中会用到，同步器实现用不到ConditionObject时可以不重写此方法。

以上模板方法在AQS的实现中都是直接抛出`UnsupportedOperationException`异常，且子类实现应当是线程安全的。


##### 1.4 *CLH* 同步队列与队列结点*Node*
在线程请求共享资源时，若共享资源此时空闲，则当前线程获得共享资源并且将共享资源设置为锁定状态。

若共享资源此时被占用而获取不到，那么此线程将会加入到CLH同步队列中阻塞自旋挂起等待。CLH队列是一个虚拟双向队列，*AQS*将每个请求共享资源的线程封装成一个CLH队列的一个结点（Node），来实现同步锁的分配。

*AQS*使用head和tail维护同步队列的头尾结点。

内部类Node封装了访问同步资源的线程，包括线程本身与线程的状态：
```java
static final class Node {
	/** 表示当前结点等待在共享模式*/
	static final Node SHARED = new Node();
	/** 表示当前结点等待在独占模式*/
	static final Node EXCLUSIVE = null;
	/** 表示线程已经被取消：等待超时或者被中断 */
	static final int CANCELLED =  1;
	/** 表示当前结点的后继结点需要被唤醒 */
	static final int SIGNAL    = -1;
	/** 结点等待在Condition上，在被signal()后会转移到同步队列 */
	static final int CONDITION = -2;

	// 共享模式： PROPAGATE只能设置在队列的头结点，表示后续结点会传递唤醒行为。
	// 在唤醒一个结点之后需要继续唤醒下一个结点的原因在于：
	// 共享模式下，若某线程能获取锁，那么下一个线程也是有可能获取到锁的，所以需要唤醒后继结点去尝试获取锁。
	// 但是若此时能正常访问锁的当前结点的ws是0则表示不需要唤醒后继结点（如没有后继结点或者没有阻塞的后继结点）。
	// 因此头结点的状态设置为PROPAGATE，表示可以传递性的，一个接一个的去唤醒后继结点来尝试获取锁。
	static final int PROPAGATE = -3;
	volatile int waitStatus; // 结点锁封装线程的状态
	volatile Node prev; // 同步队列中结点使用
	volatile Node next; // 同步队列中结点使用
	volatile Thread thread; // 结点封装的线程
	Node nextWaiter; // 条件队列中的后继结点，在条件队列中的结点使用此字段
	// ....
}
```

#### 2. 同步资源获取与释放
##### 2.1 独占方式
###### 2.1.1 获取锁acqure():
当线程调用acquire()申请获取锁资源，如果成功，则进入临界区。
当获取锁失败时，则进入一个FIFO等待队列，然后被挂起等待唤醒。
当队列中的等待线程被唤醒以后就重新尝试获取锁资源，如果成功则进入临界区，否则继续挂起等待。
```java
// 获取锁
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 尝试获取，使用子类实现的tryAcquire()方法
    	// 获取失败，添加到同步队列中自旋，竞争锁资源
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        selfInterrupt();
    // 获取成功直接返回
}

// 创建结点，并将结点入队
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试快速入队，直接CAS设置tail结点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 快速入队失败，使用CAS加自旋方式入队
    enq(node);
    // 返回新创建的结点
    return node;
}

// 结点入队，必要时初始化队列head结点，返回插入结点的前置结点
private Node enq(final Node node) {
    for (;;) { // 自旋 + CAS
        Node t = tail;
        if (t == null) { // 初始化tail结点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else { //不需要初始化的情况
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// 尝试获取资源，结点中线程自旋，争取同步锁资源
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false; // 自旋过程中中断标志
        for (;;) { // 自旋
            final Node p = node.predecessor(); // 当前结点的前置结点
            // 前置结点是头结点的时候 尝试获取同步资源
            if (p == head && tryAcquire(arg)) {
            	// tryAcquire()获取同步锁成功，设置当前结点为head
                setHead(node);
                // 指向null，破坏引用链，使之不可达，帮助GC
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }

            // 此时前置结点不是head，
            // 或者前置结点是head但是尝试获取没有获取成功。
            // 线程经parkAndCheckInterrupt()挂起被唤醒后，会继续继续进入循环尝试获取同步锁
            // 若获取到，则设置当前结点为head结点；获取不到的情况是新的线程直接获取
            // 同步资源成功，再一次进入循环去tryAcquire()
            if (shouldParkAfterFailedAcquire(p, node) && // 是否可以挂起线程
                parkAndCheckInterrupt())   // 挂起线程，线程是否中断过
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node); // 取消当前结点
    }
}
```
*shouldParkAfterFailedAcquire*()用来判断当前节点的线程尝试获取同步锁失败之后是否可以被挂起，即是否具备唤醒条件:可以挂起意味着线程挂起后可以被唤醒，这需要前置结点的状态为SINGAL。

该方法如果返回 *false*，表示当前线程还不能够被挂起，程序重新执行acquireQueued()中的循环体，进行重新判断；
如果返回true，表示当前线程可以挂起，就会执行parkAndCheckInterrupt()挂起当前线程。
```java
// 判断获取同步资源失败后是否应该挂起线程
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; //前置结点的状态
    if (ws == Node.SIGNAL) // 前置结点已设置好后继结点需要signal，故当前线程可以park
        return true; // 可以park

    //以下表示不能park当前线程，修改结点状态，并遍历地从队列中移除取消的结点
    if (ws > 0) { // 前置结点取消，向前遍历找到有效前置结点，维护同步队列结点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node; // 移除链表无效结点，更新链表结点关系
    } else { // 0 -2 -3  -2:CONDITION表示结点在条件队列中  
    	// 所以这里只能取值 0 或 PROPAGATE(-3)， 因为需要被通知signal，
    	// 故CAS修改前置结点的状态
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL); 
    }
    return false; // 线程不满足挂起的要求，需要返回以继续自旋循环处理
}

// 挂起线程并返回线程是否中断
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); //挂起线程
    // 线程唤起之后，返回中断标记，并且会清除线程的中断标记
    return Thread.interrupted();
}

// 取消结点node对同步资源的竞争，完成资源清理与队列维护
private void cancelAcquire(Node node) {
    // 结点不存在，直接返回
    if (node == null)
        return;
    node.thread = null;

    // 跳过node结点之前的已经取消的结点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // 保存node结点的前置首个结点的后继结点的引用
    Node predNext = pred.next;
	// 设置当前结点状态：取消；这样其他线程操作同步队列时，可以跳过此节点
    node.waitStatus = Node.CANCELLED;

    // node是tail，CAS移除结点。
    // 这里CAS更新链表失败不会有影响，失败表示新入队线程或者其他一个或多个取消线程已经完成链表的更新
    if (node == tail && compareAndSetTail(node, pred)) { // node为tail，则CAS更新tail为pred
    	// CAS更新tail为pred成功，即node前面的首个有效结点。
    	// 接下来CAS更新tail结点的next引用为null，期望值predNext引用的Node，
    	// CAS完成表示node节点从队列中移除
        compareAndSetNext(pred, predNext, null); 
    } else { 
        // 分两种情况：(1)node结点不是尾结点
        // (2) node结点是tail结点，但是CAS设置pred为tail结点时失败，但这种情况意味着pred
        // 在队列中节点的关系已经得到维护
        int ws; 

        // 三个条件满足则移除结点node，连接链表
        // a. pred不是head
        // b. pred状态为SIGNAL，意味着pred会通知后继结点
        // 或者ws取值0 或者CONDITION PROPAGATE，且CAS修改状态为SIGNAL成功
        // c. 而且pred封装Thread不为null
        if (pred != head && 
            ((ws = pred.waitStatus) == Node.SIGNAL || 
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&  
            pred.thread != null) {
            Node next = node.next; 
            if (next != null && next.waitStatus <= 0) 
            	// node不是tail，且其next结点有效（未被取消）需要signal，CAS修改pred的next引用，
            	// 即从队列中删除结点node，连接pred <---> next 结点
                compareAndSetNext(pred, predNext, next);
        } else { 
        	// abc三个条件不能同时满足，
        	// 表示当前结点前置结点为head结点，或者是前置不是head但结点状态取消，
            // 或者前置结点不是head且有效非取消但是pred.thread==null，
        	// 唤醒后继结点线程竞争锁资源
            unparkSuccessor(node);
        }
        // node节点next引用指向自己，一方面是表示结点不满足在条件队列的要求，
        // 因此可用于isOnSyncQueue()方法快速判断此结点在同步队列中；
        // 另外一方面也利于虚拟机进行GC。
        node.next = node; 
    }
}

// 唤醒node的后继结点
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0); //CAS清理

    // 一般情况下后继结点即为需要唤醒的结点，若后继结点取消或者为null，
    // 则从tail向前遍历，找到合适的结点将其唤醒
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
###### 2.1.2 释放锁release()：
当线程调用release()进行锁资源释放时，如果没有其他线程在等待锁资源，则释放完成。
如果队列中有其他等待锁资源的线程需要唤醒，则唤醒队列中的一个有效等待节点，通常是其后继结点。
```java
// 释放同步锁资源
public final boolean release(int arg) {
	// 调用定义在子类的释放锁逻辑tryRelease()
    if (tryRelease(arg)) {
    	// 释放成功，判断等待队列中有没有需要被唤醒的节点
        Node h = head;
        if (h != null && h.waitStatus != 0) 
        	// waitStatus不为0：表示已经不是初始状态，
        	// 需要检查后继结点情况，看是否需要唤醒
            unparkSuccessor(h); // 唤醒队列中需要唤醒的结点
        return true;
    }
    // 尝试释放锁失败 
    return false;
}
private void unparkSuccessor(Node node) { //上面调用传入node为head
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
##### 2.2共享方式
###### 2.2.1 获取共享锁acquireShared()
线程调用acquireShared()申请获取锁资源，如果成功，则进入临界区。
若获取锁失败，则创建一个共享类型的节点并进入一个 *FIFO* 等待队列，然后被挂起等待唤醒。

当队列中的等待线程被唤醒后就重新尝试获取锁资源，如果成功则唤醒后面还在等待的共享节点并把该唤醒事件传递下去，
即会依次唤醒在该节点后面的所有共享节点，然后进入临界区，否则继续挂起等待。
```java
// 申请同步锁
public final void acquireShared(int arg) {
	// 尝试获取锁资源：
	// 返回小于0的值表示获取失败；
	// 返回0表示获取成功但无可用资源；
	// 返回大于0的值表示获取成功且有可用资源
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
// 共享模式获取锁
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED); // 添加结点到同步队列
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) { //前置结点为head，意味着前置结点已经获取成功
                int r = tryAcquireShared(arg); //尝试获取锁 
                if (r >= 0) { //获取成功
                    setHeadAndPropagate(node, r); //设置头结点并传播
                    p.next = null; // help GC，清理前置结点的next引用
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return; // 获取到同步锁，返回
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) && // 是否满足线程挂起条件
                parkAndCheckInterrupt()) // 挂起线程并检查中断清理
                interrupted = true;
        }
    } finally {
        if (failed) // 获取锁失败 
            cancelAcquire(node); // 获取锁资源异常导致失败，取消结点
    }
}
// 设置head并传播
private void setHeadAndPropagate(Node node, int propagate) {
	// 保持对原head结点的引用
    Node h = head; 
    setHead(node); // 设置node为头结点

    // 共享模式下propagate > 0 
    // 或者head的后继结点需要唤醒(ws<0) 表示后继结点需要唤醒
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
        	// 共享模式，唤醒后继结点
            doReleaseShared();
    }
}
// 设置头结点
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null; 
}
// 共享模式release
private void doReleaseShared() {
    for (;;) {
        Node h = head; //取同步队列头结点
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { // 后继结点需要唤醒
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // 继续循环检查状态
                unparkSuccessor(h); // 唤醒
            }
            // 如果后继结点不需要唤醒，则应当设置为PROPAGATE，确保以后可以传递下去
            // 若是CAS修改失败，再次进入循环
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue; // loop on failed CAS
        }
        // head没有发生改变，表示设置完成，退出循环
        // head结点发生改变，表示其它线程获得了锁，因此需要再次进入循环，确保唤醒动作传递下去
        if (h == head)  // loop if head changed
            break;  
    }
}
```
###### 2.2.2 释放共享锁releaseShared()
线程调用releaseShared()释放锁资源，若释放成功，则唤醒队列中等待的节点。
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //释放资源成功
        doReleaseShared();
        return true;
    }
    // 释放资源失败
    return false;
}
// 共享模式release
private void doReleaseShared() {
    for (;;) {
        Node h = head; //取同步队列头结点
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { // 后继结点需要唤醒
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // 继续循环检查状态
                unparkSuccessor(h); // 唤醒
            }
            // 如果后继结点不需要唤醒，则应当设置为PROPAGATE，确保以后可以传递下去
            // 若是CAS修改失败，再次进入循环
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue; // loop on failed CAS
        }
        // head没有发生改变，表示设置完成，退出循环
        // head结点发生改变，表示其它线程获得了锁，因此需要再次进入循环，确保唤醒动作传递下去
        if (h == head)  // loop if head changed
            break;  
    }
}
```

#### 3. 条件队列
同步队列中获取到资源的节点即head结点，当调用*Condition*的await()方法时，将释放获取的独占资源，加入到条件队列中阻塞等待唤醒，条件队列的结点唤醒后被转移到同步队列中继续参与同步资源的竞争。

与同步队列是 *FIFO* 的双向链表不同，每一个*ConditionObject*对象维持一个单向的等待队列，所有*ConditionObject*共享同步队列。

调用signal()方法将条件等待队列中的首节点线程(或者非首节点之后的有效结点)移动到AQS同步队列尾部，并将其前继节点设置为SIGNAL或者直接唤醒线程使得被通知的线程能去获取锁。

对象的监视器模型上，一个Object拥有一个同步队列和一个等待队列；而AQS可以有一个同步队列和多个等待队列(条件队列)，一个Condition对应着一个等待队列。

ConditionObject时JUC中Condition接口的主要实现。

##### 3.1 await()方法
await()方法会使当前线程进入等待队列并且释放锁，线程转变为等待状态。 调用await()方法时，相当于同步队列的head结点移入到condition的等待队列中 。当从await()返回意味着再次获得同步锁资源，调用线程继续向下执行逻辑。

```java
public final void await() throws InterruptedException { 
	//首先检查线程中断状态，此方法会响应中断；当前线程被中断时抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter(); // 向条件队列中加入当前结点

    //释放获得的独占共享资源，并唤醒其后继结点，
    //当调用await时，说明已经进入临界区，即已经获取到独占资源：当前结点是同步队列的head结点
    int savedState = fullyRelease(node); //释放独占资源
    int interruptMode = 0;

    // 上面已经完成加入条件队列并释放独占资源的过程，
    // 接下来是阻塞等待被转到同步队列，加入到同步资源的竞争
    while (!isOnSyncQueue(node)) { // 判断是否在同步队列中：signal操作会转移结点到同步队列中
        LockSupport.park(this); // 不在sync队列，则循环挂起

        // 等待过程中线程是否被中断，中断会导致结点转到同步队列
        // 且要区别中断是发生在signal之前，还是signal之后
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break; 
    }

    // 至此，当前线程已经被加入到sync队列(由中断或者signal导致)，进行同步资源的竞争
    // acquireQueued()进行同步锁资源竞争，自旋，获得锁之后返回
    // 接下来判断获取到同步锁过程中是否线程中断
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT; // -1 

    // 删除条件队列中被取消的结点
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();

    // 抛出异常或者再次中断当前线程
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

```

await()的调用是在获取了同步锁之后，即在临界区内的操作。就像Object的wait() & notify() & notifyAll()方法，当线程获取了synchronized管理的
对象管程锁之后，进入了临界区才可以调用这些Object方法。

(1) 添加结点到条件队列：
```java
// 添加结点到条件队列
private Node addConditionWaiter() { // 条件队列添加结点
    Node t = lastWaiter;
    // 若有结点取消，则进行清理；
    // 对尾结点进行判断的原因：当释放资源出现异常时会标记尾结点的waitStatus为CANCELED
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters(); // 删除取消的结点
        t = lastWaiter;
    }
    // 创建状态为CONDITION的结点，加入到条件队列；
    // 此处的代码执行已经在临界区中，不需要做并发的控制(CAS)
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node; // 更新尾结点
    return node;
}

// 删除条件队列中取消的结点  #ConditionOBject
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) { // 从头结点开始，依次检查并删除取消的结点
        Node next = t.nextWaiter;
        // 没有等待在CONDITION上的结点，就是被取消的结点
        if (t.waitStatus != Node.CONDITION) { 
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```
(2) 判断结点是否已添加到同步队列(在while中循环)：
```java 
// 判断一个初始在条件队列中的结点是否被转移到同步队列中
final boolean isOnSyncQueue(Node node) {
	// 快速判断: waitStatus为Node.CONDITION表明其在条件队列中
	// 在同步队列中的结点显然是有前驱结点的，即prev != null
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 快速判断：如果结点有后继结点，显然其必定在同步队列中
    // 条件队列使用的是nextWaiter
    if (node.next != null) 
        return true;
    // 不符合快速判断的情况：结点一般从条件队列转到同步队列的尾部，所以从尾部查找更快一些
    return findNodeFromTail(node);
}

// 从tail开始查找node结点是否在sync队列中
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```
(3) 线程中断状态检查：
```java
// 检查等待过程中线程的中断状态：
// 1.如果线程没有中断返回0
// 2.如果线程中断，则将结点转移到同步队列
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}

// 将结点转移到同步队列，若signal先发生返回false，否则返回true
final boolean transferAfterCancelledWait(Node node) {
	// 中断导致结点转移：当在被signal()之前取消等待
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node); // CAS+自旋 入队，设置状态为0
        return true;
    }
    // 结点状态不为CONDITION表示已经被signal()通知，可能结点在入队到SYNC队列的过程
    // signal会将结点的状态设置为SIGNAL，所以上面的CAS会失败
    // 自旋等待结点转移到同步队列中，并返回false
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```
(4) 重新竞争资源：
```java
// 尝试获取资源
// 同步队列中的结点中线程自旋获取同步锁资源
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false; // 自旋过程中中断标志
        for (;;) { // 自旋
            final Node p = node.predecessor(); // 当前结点的前置结点
            // 前置结点是头结点的时候 尝试获取同步资源
            if (p == head && tryAcquire(arg)) {
            	// 获取同步锁成功，设置当前结点为head
                setHead(node);
                // 指向null，破坏引用链，使之不可达，帮助GC
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }

            // 此时前置结点不是head 或者前置结点是head但是尝试获取没有获取成功
            if (shouldParkAfterFailedAcquire(p, node) && // 是否可以挂起线程
                parkAndCheckInterrupt())   // 线程是否中断过
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node); // 取消当前结点
    }
}

// 获取同步资源失败后是否应该park线程
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; //前置结点的状态
    if (ws == Node.SIGNAL) // 前置结点已设置好后继结点需要signal，故当前线程可以park
        return true; // 可以park

    //以下表示不能park当前线程，修改结点状态换去掉取消的结点
    if (ws > 0) { // 前置结点取消，向前遍历找到有效前置结点，维护同步队列结点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else { // 0 -2 -3  -2:CONDITION表示结点在条件队列中  
    	// 所以这里只能取值 0 或 PROPAGATE(-3)， 因为需要被通知signal，
    	// 故CAS修改前置结点的状态
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL); 
    }
    return false; // 返回不应park当前线程
}

// park线程并返回线程是否中断
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```
(5) 中断处理：
```java
// 根据interruptMode判断抛出异常还是再次中断
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

##### 3.2 signal()方法
调用condition的signal()方法时，将会把等待队列的首节点开始的第一个有效结点移到等待队列的尾部，然后唤醒该节点。
signalAll()方法的区别在于它会通知条件队列上所有结点。

结点被唤醒，并不代表就会从await方法返回，也不代表该节点的线程能获取到锁，它需要加入到锁的竞争acquireQueued()方法中去，当成功竞争到锁，才能从await方法返回。
```java
public final void signal() {
	//必须是独占模式，即条件队列只适应于独占模式
    if (!isHeldExclusively()) 
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first); //唤醒
}
// 从条件队列首结点开始查找一个有效结点，将它转移到同步队列
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
// signal()操作导致的队列转换动作，返回node结点是否队列转换成功
final boolean transferForSignal(Node node) {
    // 如果不能修改修改结点状态，则表明结点已经被取消
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 从tail结点之后CAS自旋入队，此处p是node的前置结点
    Node p = enq(node); 
    int ws = p.waitStatus;
    // 如果前置结点取消，或者修改前置结点状态为SIGNAL(也就是说明当前结点在等待唤醒)失败，
    // 则直接唤醒当前的结点
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread); //唤醒当前结点的线程
    return true;
}
```
