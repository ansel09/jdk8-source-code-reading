### *ThreadLocal*
---
#### 1. 原理
每个Thread对象里有一个ThreadLocalMap字段，保存在这里面的变量只有线程自己可以访问，不会在多线程间共享，称为线程本地变量。

ThreadLocalMap定义了Entry，其key为ThreadLocal对象的虚引用，value为对应的线程本地变量实际的值。

ThreadLocalMap内部维护了Entry数组作为线程本地变量的容器，初始大小为16，可以自动扩容。

#### 2. *ThredLocalMap*源码

##### 2.1 基本结构
内部维护一个Entry数组用以存放线程本地变量。
```java
private static final int INITIAL_CAPACITY = 16; // Entry数组初始大小，扩容时都是2倍，故table的大小始终是2的幂次
private Entry[] table; // 数据容器
private int size = 0; // table中Entry数量
private int threshold; // 扩容阈值
private void setThreshold(int len) {threshold = len * 2 / 3; } //设置阈值
private static int nextIndex(int i, int len) {return ((i + 1 < len) ? i + 1 : 0); }  // 下一个数组索引值
private static int prevIndex(int i, int len) {return ((i - 1 >= 0) ? i - 1 : len - 1); } // 上一个数组索引值
```
##### 2.2 Entry
```java
// 弱引用
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value; // ThreadLocal的值
    // key为ThreadLocal对象的弱引用
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
为什么*Entry*的*key*使用弱引用的形式?
- GC发生时对象若只被弱引用关联，这个对象会被回收。
- 假若这里使用强引用，Entry中key对应的ThreadLocal对象的生命周期将会与其对应的线程相同。即只要线程存活，因为存在着这样一个对ThreadLocal的强引用，导致GC时这个对象就不会被回收。
- 基于弱引用的垃圾清理：ThreadLocalMap中使用弱引用，对于没有强引用关联的ThreadLocal对象，在GC之后Entry中的key将失效，有助于实现ThreadLocal本身的垃圾回收。ThreadLocalMap中的getEntry()和set()等方法都会调用expungeStaleEntries()方法对table中Entry的*key==null* 的过期数据进行清理。
- 在ThreadLocal不再使用时，应当及时调用其remove()方法释放相关资源。避免如下情形发生：线程池的中Thread的生命周期一般较长，若线程本地变量是一些大对象且使用后没有及时清理，可能会导致内存泄漏。

##### 2.3 构造函数
采用lazy构造的方式，*ThreadLocalMap*在第一次用到时才会创建对象并完成相应的设置。
```java
// 采用lazy构造，第一次使用时才会构造ThreadLocalMap对象，故需要传入一个第一次使用时的k-v对
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY]; // 初始化Entry数组
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1); //哈希得到数组索引
    table[i] = new Entry(firstKey, firstValue); //初始化Entry
    size = 1; //数组中节点的实际数量
    setThreshold(INITIAL_CAPACITY); //设定初始扩容阈值
}
```
##### 2.4 getEntry
*getEntry*()方法的逻辑：
- 根据key散列取模得到的index，若命中则直接返回；
- 没有直接命中则调用getEntryAfterMiss()向后线性探测，探测过程中若碰到过期数据则调用expungeStaleEntry()进行清理;
- 最后若命中则返回找到的Entry，否则返回null。
```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1); //哈希得到数组索引
    Entry e = table[i];
    if (e != null && e.get() == key) // 命中直接返回
        return e;
    else
        return getEntryAfterMiss(key, i, e); //未直接命中
}
// 未能直接命中：根据哈希得到的索引值，未能直接找到的处理逻辑
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i); //清理过期数据,即当前e=table[i]的key==null
        else
            i = nextIndex(i, len); //索引累加
        e = tab[i];
    }
    return null; //线性探测也没有找到
}
// 清理过期数据
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 清理当前slot
    tab[staleSlot].value = null; //清理线程本地变量值，以尽快GC
    tab[staleSlot] = null;
    size--; //更新table大小
    // Rehash直到遇到为null的slot停止
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;  //直到遇到slot为null停止for循环
         i = nextIndex(i, len)) { //table索引增加
        ThreadLocal<?> k = e.get(); // e = tab[i]
        if (k == null) { //对过期数据进行清理
            e.value = null; 
            tab[i] = null;
            size--;
        } else { //数据有效
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) { //若散列得到的索引并不是当前位置，重新散列
                tab[i] = null; //指向null，帮助GC，接下来进行重新散列
                while (tab[h] != null) 
                    h = nextIndex(h, len);
                // 向后线性探测找到为null的slot，将当前Entry移到这个slot存放
                tab[h] = e; 
            }
        }
    }
    // 返回staleSlot后第一个slot为null的索引值
    return i;
}
```
##### 2.5 set
*set*()方法的逻辑：
- 1.根据key的散列取模值index，在table数组中向后扫描(线性探测)，找到key则返回。
- 2.若在扫描过程中发现无效的staleSlot，对无效的staleSlot进行替换。
- 3.替换过程中会先向前扫描直到第一个空slot，若发现无效slot并标记可能的清理起点；然后再向后扫描，在向后扫描过程中查找key，若找到则与staleSlot进行替换，并判断是否需要设置当前位置为清理起点(替换之后当前slot的key为null，也就是无效的)；接下来根据起点进行扫描清理和启发式扫描清理，然后返回。
- 4.在替换过程中没有找到key，直接将值设置在staleSlot位置，同样会来进行扫描清理和启发式清理，完成后返回。此前步骤(2)(3)是为了探测staleSlot后面的位置是否因为哈希冲突的原因存在要设置的key。
- 5.若向后扫描过程中直到第一个空slot位置时，没有发现key也没有发现可替换的无效slot，则根据key和value创建新的Entry，并将它放在第一个空slot的位置。
- 6.进行启发式扫描，然后判断大小是否超过阈值，当启发式扫描有清理无效slot且大小超过阈值时，进行rehash，重新散列过程中会有全量清理和括容的动作。
```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); //由key的散列值取模得到index
    for (Entry e = tab[i];
         e != null; // e为null时循环终止
         e = tab[i = nextIndex(i, len)]) { //向后探测
        ThreadLocal<?> k = e.get(); // key
        if (k == key) { // 1.命中key，进行设置，然后返回
            e.value = value;
            return;
        }
        if (k == null) { // 2.当前slot无效，替换失效的key
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 3.新key，放在当前slot(哈希后index第一个空slot)
    // 说明上面循环没有返回，说明循环到e==null退出了for循环
    tab[i] = new Entry(key, value);
    int sz = ++size; //table更新大小
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
    	// 首先进行启发式清理：sz用于控制清理次数；i表示清理的起始位置
    	// 没有失效数据且超过size阈值，重新散列
        rehash();
}
// 替换失效的Entry
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
	// 向前扫描table，查找最靠前的一个无效slot 
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null) // key引用的ThreadLocal已经被GC
            slotToExpunge = i; // 记录无效slot
    // 向后扫描，看此前table中是否就有key
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == key) { //
        	//找到已有的位于staleSlot后面的key，交换到staleSlot处，注原先staleSlot位置的Entry是失效的
            e.value = value; //设置值
            tab[i] = tab[staleSlot]; //交换，tab[i]处存放失效Entry
            tab[staleSlot] = e;
            // 若向前扫描没有找到失效的slot，则更新slotToExpunge标记为当前i，从当前位置开始清理
            if (slotToExpunge == staleSlot) 
                slotToExpunge = i;
            // 两次清理：先普通清理，再启发式清理，其中启发式清理可能会多次调用普通清理方法
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len); 
            return;
        }
        // 当前slot无效，且向前扫描没有找到失效的slot，更新清理起点为当前值(i)
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }
    // table中没有key，将新Entry放到staleSlot处
    tab[staleSlot].value = null;  //置为null
    tab[staleSlot] = new Entry(key, value); 
    // 探测过程中若发现无效的Entry，进行清理
    if (slotToExpunge != staleSlot) //即向前扫描发现失效的slot
        // 两次清理：先普通清理，再启发式清理，其中启发式清理可能会多次调用普通清理方法
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}

// 启发式扫描清理：Heuristically scan
// 清理过期失效Entry，若有清理动作返回true；
// set()方法中、和set()方法调用的replaceStaleEntry()会调用此方法；
// 启发式清理会调用清理方法expungeStaleEntry()
private boolean cleanSomeSlots(int i, int n) {
	// 1.n控制扫描次数，若循环中没有发现失效slot，则循环log2(n)次结束；
	// 2.若循环中发现了失效slot，将n重置为tab.length继续扫描；
	// 每次从一个空slot开始扫描
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) { 
        	// 过期无效数据，需要清理；重置n即重置循环次数log2(n)
            n = len;
            removed = true; // 标记循环中有清理动作
            //进行清理，并返回当前staleSlot之后的第一个空slot，用于继续循环
            i = expungeStaleEntry(i); 
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}

// 重新散列：首先全量清理重新散列，若超过阈值扩容并重新散列
private void rehash() {
    expungeStaleEntries(); //全量扫描清理
    // 经过清理之后，size会变小，因此使用比默认阈值更小的阈值进行判断
    // 采用阈值：2/3 * len * 3/4 =  1/2 len 
    if (size >= threshold - threshold / 4)
        resize();
}
// 扩容，2倍旧的容量
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
            	// 键有效
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null) // 若有位置冲突，进行线性探测，找到空slot
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++; //扩容table的size计数
            }
        }
    }
    // 扩容后更新参数和引用
    setThreshold(newLen);
    size = count;
    table = newTab;
}
// 全量清理
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

##### 2.6 remove
```java
// 根据key进行移除
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) { // 执行先后扫描动作
            e.clear(); // 清理：找到key，将key=null
            expungeStaleEntry(i); //扫描清理
            return;
        }
    }
}
```
#### 3. *ThradLocal*
ThradLocal对外提供的方法都是根据当前调用线程，得到线程的ThreadLocalMap，然后对其进行set/get/remove等操作。
##### 3.1 哈希函数
```java
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode(); //每个ThradLocal实例在初始化时完成设置
    private static AtomicInteger nextHashCode = new AtomicInteger(); //静态变量，即ThradLocal类所拥有
    //基于理论和实践，使用魔数0x61c88647每次累加的值对2^n取模，能够得到良好的散列值。
    private static final int HASH_INCREMENT = 0x61c88647;
    //因为nextHashCode为类变量，所以每个ThreadLocal实例在初始化时threadLocalHashCode的取值都在现有的值基础上累加魔数值。 
    private static int nextHashCode() {return nextHashCode.getAndAdd(HASH_INCREMENT); }
    //更多...此处省略...
}
```
哈希冲突的解决
- 在逻辑上Entry[]数组索引是环形的，*ThreadLocalMap*使用线性探测法来解决散列冲突。

##### 3.2 get()
```java
public T get() {
    Thread t = Thread.currentThread(); //调用线程
    ThreadLocalMap map = getMap(t); //获取map
    if (map != null) { //线程之前已初始化ThreadLocalMap
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 首次使用线程的ThreadLocalMap，进行构造
    return setInitialValue();
}
// 初始化设置
private T setInitialValue() {
    T value = initialValue(); // initialValue() returns null.
    Thread t = Thread.currentThread(); 
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
void createMap(Thread t, T firstValue) {
	// 调用构造函数
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

```
##### 3.3 set()
```java
// 设置value
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); //获取当前线程的ThreadLocalMap
    if (map != null)
        map.set(this, value); //ThreadLocal#set()
    else // 首次使用当前线程的ThreadLocalMap，需要创建
        createMap(t, value);
}
```
##### 3.4 remove()
```java
public void remove() {
	ThreadLocalMap m = getMap(Thread.currentThread());
 	if (m != null)
    	m.remove(this);
}
```
##### 3.5 关于InheritableThreadLocal
*InheritableThreadLocal* 继承ThreadLocal，提供了一种父子线程共享，子线程可以根据父线程的线程本地变量进行构造。下面是ThreadLocalMap中相应的方法：
```java
// 创建Map
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
// ThreadLocalMap中根据传入父线程的map进行构造
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```