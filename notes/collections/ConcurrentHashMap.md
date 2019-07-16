### *ConcurrentHashMap源码分析*
---
#### 1.概述

+ HashMap是非线程安全的，在JDK1.5之前，通常使用HashTable作为HashMap的线程安全版本使用。
+ JDK1.7是基于Segement分段锁和Node实现的。DEFAULT_CONCURRENCY_LEVEL=16表示默认的并发度，表示CHM维护了多少个分段锁Segement，Segement继承自ReentrantLock；在定位的时候会进行两次哈希，第一次定位到哪一把分段所，第二次哈希则定位桶...
+ JDK1.8基于Node + CAS + synchronized实现，不再使用JDK1.7的设计。

#### 2.常量，属性


```java
/* ---------------- Constants -------------- */
private static final int MAXIMUM_CAPACITY = 1 << 30;
private static final int DEFAULT_CAPACITY = 16;
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
private static final int DEFAULT_CONCURRENCY_LEVEL = 16; //默认并发度
private static final float LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8; //树化阈值
static final int UNTREEIFY_THRESHOLD = 6; //树转换成链表阈值
static final int MIN_TREEIFY_CAPACITY = 64; //树化的前提，容量达到64+

/**
 * Minimum number of rebinnings per transfer step. Ranges are
 * subdivided to allow multiple resizer threads.  This value
 * serves as a lower bound to avoid resizers encountering
 * excessive memory contention.  The value should be at least
 * DEFAULT_CAPACITY.
 */
private static final int MIN_TRANSFER_STRIDE = 16;

/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static int RESIZE_STAMP_BITS = 16;

/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

/*
 * Encodings for Node hash fields. See above for explanation.
 */
// 表示这是一个ForwardingNode
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
// 
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

/** Number of CPUS, to place bounds on some sizings */
// CPU逻辑核心数
static final int NCPU = Runtime.getRuntime().availableProcessors();
````


#### 3.结点相关的内部类
ConcurrentHashMap中定义了几个结点相关的内部类。



#### 4.构造方法


#### 3.重要方法

##### 3.1 put方法
```java
// 
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0; //记录结点元素个数，用于控制扩容或者转移为树
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0) // 首次插入结点，初始化table
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 待放入位置为空
            if (casTabAt(tab, i, null, 
                         new Node<K,V>(hash, key, value, null))) // CAS尝试设置value
                break;                   // CAS设置成功，退出循环，这种情况不需要加锁
        }
        else if ((fh = f.hash) == MOVED) //扩容，正在复制数组
            tab = helpTransfer(tab, f); // 帮助复制数组
        else {
            V oldVal = null;
            // 散列位置有值，加锁： 判断是RB树和还是链表
            synchronized (f) { // 取得散列位置的结点的内置锁
                if (tabAt(tab, i) == f) { // 
                    if (fh >= 0) { // (1) fh>0 说明是链表
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val; // 记录旧值
                                // key已存在，替换
                                if (!onlyIfAbsent) 
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) { //遍历到链表尾部，添加结点，保存K-V值
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // (2) 红黑树
                    else if (f instanceof TreeBin) { // 散列位置存放的是红黑树头结点，TreeBin类型结点
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) { // 链表是否需要转换成红黑树
                if (binCount >= TREEIFY_THRESHOLD) 
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}


```



#### 4.迭代器
#### 5.总结







```java
/* ---------------- Nodes -------------- */

/**
 * Key-value entry.  This class is never exported out as a
 * user-mutable Map.Entry (i.e., one supporting setValue; see
 * MapEntry below), but can be used for read-only traversals used
 * in bulk tasks.  Subclasses of Node with a negative hash field
 * are special, and contain null keys and values (but are never
 * exported).  Otherwise, keys and vals are never null.
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    // 不支持修改，不可变
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**
     * Virtualized support for map.get(); overridden in subclasses.
     */
    Node<K,V> find(int h, Object k) { // 哈希值h
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}


/** 根据sizeCtl初始化数组table
 *  1.线程尝试竞争table初始化任务，竞争失败则等待其它线程完成初始化，返回。
 *  2.竞争成功，则开始初始化table，并设置扩容阈值。
*/
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0) // sizeCtl < 0 表示已经有其它线程在在初始化table
            // 当前线程竞争初始化任务失败，自旋直到其它线程完成对table初始化，然后退出while循环
            Thread.yield(); 

        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // CAS尝试设置sizeCtl为-1
            try { //设置sizeCtl成功，说明线程竞争得到初始化任务
                if ((tab = table) == null || tab.length == 0) { // table未初始化
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY; // sc保留了开始的sizeCtl
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; // 初始化table
                    table = tab = nt; 
                    sc = n - (n >>> 2); // 初始化后设置扩容阈值为 3/4 * n
                }
            } finally {
                sizeCtl = sc; // 设置下次扩容阈值
            }
            break;
        }
    }
    return tab;
}
```





```java
/* ---------------- Conversion from/to TreeBins -------------- */

/**
 * Replaces all linked nodes in bin at given index unless table is
 * too small, in which case resizes instead.
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
    	// 表大小小于64，先扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);

        // table大小超过64，链表转换红黑树
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) { //遍历链表
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null); 
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}

/**
 * Returns a list on non-TreeNodes replacing those in given list.
 */
// 红黑树 --> 链表
static <K,V> Node<K,V> untreeify(Node<K,V> b) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = b; q != null; q = q.next) {
        Node<K,V> p = new Node<K,V>(q.hash, q.key, q.val, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```