### *HashMap源码分析*
---


#### 1.概述
+ JDK1.7中HashMap基于数组 + 链表的设计，在JDK1.8中进行了优化，改用数组 + 链表/红黑树的设计，当桶中链表元素过长(当达到8)时链表会转换为红黑树，同时当红黑树结点过少(当减至6)时会转换为链表，以此来提高HashMap结点的查找定位速度。

+ 定义了两个内部类：(1) 基本的哈希表结点Node，实现了Map中的Entry接口，桶中结点较少时用Node组成一个单向链表；(2) 红黑树结点TreeNode，继承LinkedHashMap.Entry，而LinkedHashMap.Entry继承自HashMap.Entry，提供了关于红黑树的相关操作：为保证红黑树特性的旋转调整操作-左旋右旋，以及添加元素，树化，转换为链表等操作。

#### 2.属性，构造函数与hash函数
##### 2.1 HashMap中定义的几个静态常量和属性
```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //默认初始容量值
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f; //默认负载因子
static final int TREEIFY_THRESHOLD = 8; //链表转换为红黑树的阈值
static final int UNTREEIFY_THRESHOLD = 6; //红黑树转换回链表的阈值
// 桶中链表转换红黑树的前提：容量超过此阈值64，
// 否则即使满足转换为红黑树，也需先扩容
static final int MIN_TREEIFY_CAPACITY = 64; 

// 桶数组，并不采用默认机制对该数组序列化
transient Node<K,V>[] table; 
transient Set<Map.Entry<K,V>> entrySet;
// HashMap中的结点数量，也就是元素个数
transient int size;
// 修改次数，结构性：增删，用于快速失败机制fail-fast
transient int modCount;

// 下次扩容的阈值 threshold = capacity * loadFactor
int threshold;
// 负载因子，表明桶数组的使用情况，可以大于1
final float loadFactor;

/* 此外还有两个定义在父类AbstractMap中的字段 */
transient Set<K>        keySet;
transient Collection<V> values;
```

###### 2.2 构造方法
HashMap中有三个构造方法，从构造方法来看，除传入Map带参构造方法外，其余的构造方法只是设置了初始容量和负载因子这两个字段的值，并没有对底层的Node数组进行初始化，采取的是在第一次插入元素的时候初始化，申请相应大小的内存空间。

为什么这么做？针对创造HashMap对象却没有使用的情景，首次插入元素时初始化底层Node数组的方式，可以节省不必要的内存开销。
```java
// 1.默认无参构造函数：使用16和0.75
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
// 2.设置初始化容量，使用默认负载因子0.75
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 3.带参构造方法：初始容量和负载因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //设置扩容阈值为不比传入值小的最小2的幂次
    this.threshold = tableSizeFor(initialCapacity); 
}

// 4.传入Map构造对象
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
##### 2.3 tableSizeFor方法
关于tableSizeFor方法，是根据传入值initialCapacity，得到一个不比initialCapacity小的最小2^n。tableSizeFor方法一般用于设置HashMap的容量，为什么容量要为2的幂次？为了提高计算效率，因为取模的计算效率远不如位运算，当桶数组长度为len = 2^n，则满足hash % len = hash & (len -1)，可以提升桶定位的效率。下面是tableSizeFor方法的实现：
```java
// 通过一系列位运算，得到满足条件的2的幂次值
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

##### 2.4 hash函数

哈希函数取自key的哈希值h和h右移16位的异或运算的结果，为什么要这样做？HashMap一开始的容量值一般比较小，比如无参构造方法中容量值默认为16，如果直接使用哈希值故在做hash & (n - 1)散列时只会用到哈希值得低位部分，采用hash = h ^ (h >>> 16)之后，使得高位也参与哈希，从而更大程度上减少哈希碰撞。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### 3.重要方法 
##### 3.1 put相关方法
put相关方法的核心方法是putVal()，其逻辑大致如下：

+ (1) 首先检查底层数组table是否为空，若为空，则调用热resize方法进行扩容.
+ (2.1) 根据key计算索引，定位到的桶中没有元素，则直接放入桶中.
+ (2.2) 若定位到的桶中有元素，则分命中桶中第一个元素，以及没有命中第一个元素而首节点是Node和TreeNode共三种情况，分别进行定位，没有找到key直接插入新节点，找到key则进行标记获得引用；后续判断之后再决定是否需要替换为新元素.
+ (3) put操作完成之后，更新modCount和size，并判断需要扩容.

```java
// 对外提供的方法
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}

/**
 * HashMap添加元素的核心方法实现
 * @param onlyIfAbsent true表示若key存在，则不会替换其值
 * @param evict if false, the table is in creation mode.若false，表示底层桶数组处于创建模式
 * @return 返回旧值
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1.检查table是否为空，为空则调用resize()进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2.1 根据key计算索引，若桶中没有存放元素，创建结点直接放入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 2.2 桶中已有元素
    else {
        Node<K,V> e; K k;
        // 2.2.1命中桶中首元素，标记，后续会视情况而替换其值
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 2.2.2 桶中元素存放结构为红黑树，在红黑树中定位，若找到key存在则标记；
        // 否则在合适位置插入新节点TreeNode
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 2.2.3在链表中定位结点位置
        else {
        	// binCount用于对该桶链表结点数量进行统计
            for (int binCount = 0; ; ++binCount) {
            	// 2.2.3.1遍历到链表尾部，仍然没有key存在，则添加到尾部
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 达到树化阈值，将链表转化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st 超
                        treeifyBin(tab, hash);
                    break;
                }
                // 2.2.3.2 遍历链表过程中，找到key已经存在，标记
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { //若已经存在相应的key，视情况进行替换
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null) //进行替换，包括旧值null的情况
                e.value = value;
            // 放入元素完成后的操作，这里是个空函数
            // 在LinkedHashMap中会用此函数维护结点插入顺序：move node to last
            afterNodeAccess(e); 
            return oldValue;
        }
    }
    // 3.更新modCount和size，并判断是否需要扩容
    ++modCount;
    if (++size > threshold)
        resize();
    // 回调函数，用于LinkedHashMap
    afterNodeInsertion(evict);
    return null;
}
```
##### 3.2 remove方法
删除结点：首先定位桶位置，若找到结点，标记也就是得到其引用，后续会删除结点，并更新modCount和size.
```java
// 1.根据key删除结点，返回被删除节点
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
// 2.根据key和value删除结点
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}
// 删除结点操作的核心方法removeNode
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 1. 桶定位之后，查找key是否存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
    	// 1.1 命中桶内第一个元素，标记，用node存储引用
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
        	// 1.2 桶内结点结构为红黑树，调用相应的方法查找结点，若没有找到会返回null
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            // 1.3 桶内为结点结构为链表，遍历，在链表中查找key
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 2.若找到key相应的结点，按不同的数据结构，删除
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
        	// 在红黑树中删除结点
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 在链表中删除结点，更新结点的next指向
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            // 更新modeCount和size
            ++modCount;
            --size;
            afterNodeRemoval(node); //用于LinkedHashMap，此处是个空函数
            return node;
        }
    }
    return null;
}
```

##### 3.3 get方法
get方法比较简单，根据key查找value.
```java
// 根据key获取value
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 首先桶定位
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
    	// 检查桶内第一个结点，是否命中
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶内还有其它结点，则分桶内结构是红黑树还是链表
        if ((e = first.next) != null) {
        	// 结构为红黑树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 结构为链表
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    // 没有找到相应结点，返回null
    return null;
}
```
##### 3.4 扩容，resize()方法
扩容发生在首次往哈希表中添加元素或哈希表中元素个数达到扩容阈值之时.

+ 首先会计算得到新容量值，与之同时计算新扩容阈值，之后初始化新底层数组.

+ 接下来若非首次添加元素扩容的情形，则需要复制原数组元素到新底层数组，遍历原底层数组，将原数组桶内元素复制到新数组：
	
	- 桶内只有一个元素，直接复制引用到新数组桶位置.
	- 桶内还有其它元素：若桶内结构为红黑树，则拆分红黑树为两颗树，若树太小则转化为链表，最后分别复制引用到新数组的两个桶中；若桶内结构为链表，则将链表拆分为二，放到新数组的两个桶中.

```java
// 扩容：	 1.初始扩容； 2. 非初始扩容，2倍
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table; //原底层数组
    int oldCap = (oldTab == null) ? 0 : oldTab.length; //原容量
    int oldThr = threshold; //原扩容阈值
    int newCap, newThr = 0;
    // 1.计算新容量，新扩容阈值
    // 1.1 原容量>0，即非HashMap初始化的情况
    if (oldCap > 0) { 
    	//当超出设置的容量上限值时，不再进行扩容
        if (oldCap >= MAXIMUM_CAPACITY) { 
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 计算新容量值：2倍旧容量，且旧容量要大于等于默认初始值16
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 相应地，扩容阈值*2
    }
    // 1.2. 程序能走到这，原容量为0，但原扩容阈值>0，
    // 说明原扩容阈值是在调用HashMap(initCapcity)或者HashMap(initCapacity,loadFactor)
    // 构造对象时设置的，即现在是HashMap首次添加元素时所进行的扩容动作
    else if (oldThr > 0) 
        newCap = oldThr;
    // 1.3.原容量为0，原扩容阈值也为0，即对象通过无参构造方法构造，这里也是
    // 首次添加元素的情况
    else {               
        newCap = DEFAULT_INITIAL_CAPACITY; // 默认容量大小16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//12
    }

    // 若新扩容阈值为0，根据cap*loadFactor重新计算，并需要判断边界
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr; //更新扩容阈值字段和底层数组为新值
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;


    // 2.非首次扩容，原底层数组有结点不为空，复制元素
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 当前桶非空
            if ((e = oldTab[j]) != null) { 
                oldTab[j] = null; //原底层数组清理
                // 2.1 只有一个结点，直接重新散列到新底层数组
                if (e.next == null) 
                    newTab[e.hash & (newCap - 1)] = e;
                // 2.2 桶内为红黑树
                // 拆分红黑树为2个树，若树太小会转换为链表
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 2.3 桶内为链表
                else { // preserve order
                	// 双指针用法：head & tail, 初始指向相同
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 接下来将链表中结点一分为二，
                        // 当然有可能其中一个为空链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 一个链表放在桶数组原位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 另外一个链表放在桶数组往后oldCap处的位置
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 最后返回扩容之后的底层数组
    return newTab;
}
```
##### 3.5 序列化
HashMap并没有使用默认的序列化机制，其重写了writeObject和readObject方法，主要是重新定义了底层Node数组的序列化和反序列化逻辑.

为什么要重写？首先一般情况下HashMap放不满，故重新计算容量，创建合适大小的底层数组以合理利用内存资源；此外，不同JVM下使用的native方法hashCode可能不一样，相同的key散列位置可能会不一致，所以直接使用默认机制序列化底层数组table可能会引起错误.
```java
// 1.序列化
private void writeObject(java.io.ObjectOutputStream s)
    throws IOException {
    int buckets = capacity(); //桶数量

    // 序列化采用默认机制的字段
    // Write out the threshold, loadfactor, and any hidden stuff
    s.defaultWriteObject();
    s.writeInt(buckets); //序列化桶数量
    s.writeInt(size); //实际元素值
    internalWriteEntries(s);
}
// 获取容量
final int capacity() {
	//若底层数组为null则取扩容阈值(有参构造)或者默认容量值(无参构造)
    return (table != null) ? table.length :
        (threshold > 0) ? threshold : //扩容中不为0，有参构造所致
        DEFAULT_INITIAL_CAPACITY; //无参构造，扩容阈值0，因为构造时没有初始化该字段
}
// 序列化哈希表中的Entry
void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
    Node<K,V>[] tab;
    if (size > 0 && (tab = table) != null) { //遍历桶数组
    	//序列化key和value，根据key和value的具体类型完成它们的序列化：默认的 or 重写的
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                s.writeObject(e.key); 
                s.writeObject(e.value);
            }
        }
    }
}

// 2.反序列化
private void readObject(java.io.ObjectInputStream s)
    throws IOException, ClassNotFoundException {
    // 先反序列化为采用默认序列化机制的字段
    s.defaultReadObject();
    reinitialize();
    // 参数检查
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new InvalidObjectException("Illegal load factor: " + loadFactor);
    s.readInt();                // Read and ignore number of buckets 桶长度，这里忽略
    int mappings = s.readInt(); // Read number of mappings (size)
    if (mappings < 0) //检查
        throw new InvalidObjectException("Illegal mappings count: " + mappings);
    else if (mappings > 0) { // (if zero, use defaults)
        // Size the table using given load factor only if within
        // range of 0.25...4.0
        float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f); //可取范围：0.25 ~ 4
        float fc = (float)mappings / lf + 1.0f;
        // 计算新容量值
        int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                   DEFAULT_INITIAL_CAPACITY :
                   (fc >= MAXIMUM_CAPACITY) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor((int)fc));
        float ft = (float)cap * lf;
        threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                     (int)ft : Integer.MAX_VALUE);
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] tab = (Node<K,V>[])new Node[cap]; 
        table = tab;

        // 读取元素数据，放到哈希表中
        for (int i = 0; i < mappings; i++) {
            @SuppressWarnings("unchecked")
                K key = (K) s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V) s.readObject();
            putVal(hash(key), key, value, false, false);
        }
    }
}

// 重初始化，即重置字段为零状态
void reinitialize() {
    table = null;
    entrySet = null;
    keySet = null;
    values = null;
    modCount = 0;
    threshold = 0;
    size = 0;
}
```
##### 3.6 克隆
HashMap的克隆方法是会返回对象的一份浅拷贝.
```java

public Object clone() {
    HashMap<K,V> result;
    try {
        result = (HashMap<K,V>)super.clone();
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
    result.reinitialize(); //重新初始化，重置相关字段状态
    result.putMapEntries(this, false); //重新放入元素
    return result;
}
```

#### 4.迭代器
##### 4.1 内部类EntrySet，KeySet，Values
这三个内部类的方法都是对HashMap底层数据的操作，一种简单的逻辑封装；在HashMap中可以通过entrySet()，keySet()，values()方法获得其对象，查看源码可见，若HashMap对象字段entrySet/keySet/values均为null，则进行初始化，考虑到多线程情况，可能会有两个线程得到的不是同一个对象实例。
```java
// 用于键值对
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }

    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    public final Spliterator<Map.Entry<K,V>> spliterator() {
        return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

// 用于HashMap的一些key，封装了对key访问的一些操作
final class KeySet extends AbstractSet<K> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    // 返回迭代器
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super K> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

// 用于HashMap的value
final class Values extends AbstractCollection<V> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    // 返回迭代器
    public final Iterator<V> iterator()     { return new ValueIterator(); }
    public final boolean contains(Object o) { return containsValue(o); }
    public final Spliterator<V> spliterator() {
        return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

##### 4.2 迭代器KeyIterator，ValueIterator，EntryIterator

为HashMap中的三个内部类KeySet，Values，EntrySet三个集合类所提供的迭代器，均继承抽象父类HashIterator，此类中提取了一些公用的方法；三个迭代器类KeyIterator，ValueIterator，EntryIterator各自实现next方法。

KeySet，Values，EntrySet三个内部类可用于遍历HashMap中的数据。

HashMap也是支持快速失败机制的，并发修改时会导致抛出并发修改失败异常ConcurrentModificationException。

```java
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot 当前桶的位置

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
        	// 移动到第一个非空的桶
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }
    // 下一结点
    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;

        // 快速失败机制
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
        	// 移动，跳过两个非空桶之间的空桶
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e; //返回下一结点的引用
    }

    // 迭代时删除结点
    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null; // 删除
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount; //更新expectedModCount
    }
}

final class KeyIterator extends HashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().key; }
}

final class ValueIterator extends HashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```
可以通过entrySet()，values()和keySet()方法获取相应的字段值。


#### 5.总结
+ HashMap在JDK1.8中做的优化
+ Fail-fast机制
+ 扩容：初始扩容和非初始扩容
+ 其键和值均可以为null，而HashTable和ConcurrentHashMap的键值是不能为null的