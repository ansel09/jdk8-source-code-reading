### *CopyOnWriteArrayList源码分析*
---

#### 1.概述 
+ 与ArrayList相比，CopyOnWriteArrayList是线程安全的容器；
+ CopyOnWriteArrayList遍历时不需要加锁，不会抛出ArrayList会抛出的并发修改异常；
+ 基于写时复制的策略，底层通过复制数组的方式来实现.

#### 2.*COW* 机制

*Copy on write，COW*即写入时复制，其指的是如果多个调用者同时请求相同资源，它们会获取相同的指针，这意味着它们访问的是相同的资源；直到某个调用者试图修改资源的内容时，系统才会真正复制一份副本给该调用者使用，而所有其它调用者访问的仍然是一开始拿到的那同一份资源。

这种策略在没有调用者修改资源的情形下，系统就不会有创建副本，多个调用者的读取操作共享同一份资源。

写入时复制机制在处理过程中需要维持一个读请求使用的指针，在新数据写入完成后更新指针，以提升读写并发能力。COW也需要要提供数据更新也就是指针替换过程中的原子性。

JDK中使用的场景是并发容器:CopyOnWriteArrayList，CopyOnWriteArraySet。

#### 3.重要方法
##### 3.1 构造方法
在分析构造方法之前，先看一下CopyOnWriteArrayList的几个属性：
```java
// 使用了重入锁，保证线程安全
final transient ReentrantLock lock = new ReentrantLock();

// Unsafe指针与锁偏移，用于反序列化时恢复锁
private static final sun.misc.Unsafe UNSAFE;
private static final long lockOffset;

// 内部的数据容器，只能通过getArray()和setArray()方法访问
// volatile保证了其线程间的可见性
private transient volatile Object[] array;
final Object[] getArray() { return array; }
final void setArray(Object[] a) { array = a; }
```
与ArrayList一样，CopyOnWriteArrayList的构造方法有三个:

```java

// 1.无参构造函数，比较简单，初始化array为空数组
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}

// 2.有参构造函数，初始化array为传入数组toCopyIn的副本
// 创造的List对象将持有数组副本
public CopyOnWriteArrayList(E[] toCopyIn) {
	// Arrays.copyOf()中会调用System.arrayCopy()方法，创建副本
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

// 3.有参构造函数，根据传入集合初始化array，创建对象
// 若入参c为null会抛出NullPointerException
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    // 入参是CopyOnWriteArrayList实例
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        // 此Bug在ArrayList中也有提及，
        // 因此若此处返回的不是Object[]，使用Arrays.copyOf方法复制一下，将返回Object[]
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    // 设置array
    setArray(elements);
}
```

##### 3.2 add方法
add方法有2个：在链表尾部添加元素和在指定位置添加元素。
```java
// 1. 在尾部添加元素
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock(); //尝试获取锁
    try {
    	// 获取到锁返回后，执行下面代码
        Object[] elements = getArray();
        int len = elements.length;
        // 写时复制，创建新数组
        Object[] newElements = Arrays.copyOf(elements, len + 1); 
        newElements[len] = e;
        setArray(newElements); // 写时复制：写操作完成时，替换底层数组为新数组
        return true;
    } finally {
        lock.unlock(); //释放锁
    }
}

// in Arrays...
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
// in Arrays...重载的copyOf方法
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength] //是Object[]类型
        // Array.newInstance 最终会调用native方法newArray创建指定元素类型的数组
        : (T[]) Array.newInstance(newType.getComponentType(), newLength); 

    // 将元素复制到新数组并返回新数组引用
    System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));
    return copy;
}


// 2. 在指定位置添加元素
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray(); // 保持原数组的引用
        int len = elements.length;
        if (index > len || index < 0) // 边界检查
            throw new IndexOutOfBoundsException("Index: "+index+ ", Size: "+len);

        Object[] newElements; // 声明新数组
        int numMoved = len - index; // 需要移动的元素个数，逻辑右移一个索引的元素个数
        if (numMoved == 0) //不需要移动，在尾部添加元素
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1]; 

            //初始化新数组之后，接下来复制位于插入位置的左右两侧区间的元素
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1, numMoved);
        }
        newElements[index] = element;
        setArray(newElements); //写时复制，写操作完成后，替换底层数组
    } finally {
        lock.unlock();
    }
}
```

向CopyOnWriteArrayList中添加元素，就需要增加数组长度，与ArrayList的扩容机制不同，从源码中可以看出每次添加元素时创建新数组，底层新数组长度比原先数组的大待添加元素的个数，这里只添加一个元素，所以新数组仅仅会大1；最后基于写时复制的策略，在写入完成也就是元素添加之后替换掉原先底层的数组。


addIfAbsent方法对于入参为链表中已存在的元素，则不予添加到链表中。
```java
// 判断元素是否已经存在链表中，不存在添加到链表中，存在则返回false
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray(); //原底层数组
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}

private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock(); //竞争锁
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) { //传入的底层数组快照与当前数组不同
        	// 表示有其它线程先获得锁，完成了底层数组的写入替换操作
            // 取两者大小较小者的长度作为底层数组的检查部分
            int common = Math.min(snapshot.length, len);

            // 若在底层数组下标[0,common-1)的部分中，已经存在
            // 待添加元素e，则返回fasle
            // 这里的意思是判断有没有其它线程已经把当前待添加元素添加到链表中
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i])) 
                    return false;

            //若下标区间[common,len)底层数组中已经有待添加元素，返回false
            if (indexOf(e, current, common, len) >= 0) 
                    return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements); //写完替换底层数组
        return true;
    } finally {
        lock.unlock();
    }
}
```
addAll添加传入集合中的元素，方法有三个：(1)从尾部添加所有元素； (2)从指定位置添加所有元素； (3)从尾部添加指定集合中不与现存元素重复的元素。

```java
// 1. 添加传入集合中的所有元素	
public boolean addAll(Collection<? extends E> c) {
    Object[] cs = (c.getClass() == CopyOnWriteArrayList.class) ?
        ((CopyOnWriteArrayList<?>)c).getArray() : c.toArray();
    if (cs.length == 0) // 检查集合非空
        return false;
    final ReentrantLock lock = this.lock; 
    lock.lock(); // 获取锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //原先链表中没有数据，直接设置底层数组array
        if (len == 0 && cs.getClass() == Object[].class) 
            setArray(cs);
        else {
        	// 为新数组申请一块新的内存空间，大小为原数组大小+待添加集合元素个数
            Object[] newElements = Arrays.copyOf(elements, len + cs.length);
            System.arraycopy(cs, 0, newElements, len, cs.length);
            // 写完替换
            setArray(newElements);
        }
        return true;
    } finally {
        lock.unlock();
    }
}

// 2. 从指定位置添加传入集合中的元素
public boolean addAll(int index, Collection<? extends E> c) {
    Object[] cs = c.toArray();
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0) //边界检查
            throw new IndexOutOfBoundsException("Index: "+index+ ", Size: "+len);
        if (cs.length == 0)
            return false;
        int numMoved = len - index; //计算需要移动的元素个数
        Object[] newElements; 
        if (numMoved == 0) // 不需要移动元素
            newElements = Arrays.copyOf(elements, len + cs.length);
        else { //需要移动
            newElements = new Object[len + cs.length];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + cs.length, numMoved);
        }
        System.arraycopy(cs, 0, newElements, index, cs.length);
        setArray(newElements); //写完替换
        return true;
    } finally {
        lock.unlock();
    }
}

// 3.对于传入集合的元素，仅仅添加链表中原先不存在元素
public int addAllAbsent(Collection<? extends E> c) {
    Object[] cs = c.toArray();
    if (cs.length == 0)
        return 0;
    final ReentrantLock lock = this.lock;
    lock.lock(); //获取锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        int added = 0;
        // 遍历传入集合中的所有元素，压缩数组，过滤掉重复元素
        for (int i = 0; i < cs.length; ++i) {
            Object e = cs[i];
            if (indexOf(e, elements, 0, len) < 0 && // 判断是链表中是否存在当前元素
                indexOf(e, cs, 0, added) < 0) //是否与之前遍历的元素重复
                cs[added++] = e;//复用数组cs的内存空间，手法值得借鉴
        }
        if (added > 0) { //有新元素
            Object[] newElements = Arrays.copyOf(elements, len + added);
            System.arraycopy(cs, 0, newElements, len, added);
            setArray(newElements); //写完替换底层数组
        }
        return added;
    } finally {
        lock.unlock();
    }
}
```

##### 3.3 remove方法
基本的删除包括删除指定位置的元素和删除指定元素。
```java
// 1.删除指定位置的元素
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1; //需要移动元素的个数
        if (numMoved == 0) //无需移动直接复制
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1]; //申请新数组
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index, numMoved);
            setArray(newElements); //写完替换底层数组
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

// 2.删除指定元素
public boolean remove(Object o) {
    Object[] snapshot = getArray(); //快照
    //找到元素的位置
    int index = indexOf(o, snapshot, 0, snapshot.length); 
    return (index < 0) ? false : remove(o, snapshot, index);
}
// 删除指定元素的具体逻辑实现
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock(); //竞争锁
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) findIndex: { // 其它线程有修改链表结构
            int prefix = Math.min(index, len);
            // 其它线程有修改链表结构，判断待删除元素在区间[0,prfix)是否还存在
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i; //还存在，更新其当前的index
                    break findIndex;
                }
            }
            // 到这，说明待删除元素在数组下标区间[0.prefix)不存在
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            // 继续判断后面的区间是否存在待删除的元素
            index = indexOf(o, current, index, len);
            if (index < 0) //不存在直接返回
                return false;
        }
        // 存在待删除元素，进行删除动作
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        setArray(newElements); //写操作完成后替换底层数组
        return true;
    } finally {
        lock.unlock();
    }
}
```
还有从链表中删除与传入集合相交的元素，删除指定位置范围的元素，以及删除符合条件的元素。

```java
// 1.删除与传入集合相交的元素
public boolean removeAll(Collection<?> c) {
    if (c == null) throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock(); //竞争锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (len != 0) {
            // temp数组保存的是需要保留的元素
            int newlen = 0;
            Object[] temp = new Object[len];
            for (int i = 0; i < len; ++i) {
                Object element = elements[i];
                if (!c.contains(element))
                    temp[newlen++] = element;
            }
            if (newlen != len) { //有元素被删除
                setArray(Arrays.copyOf(temp, newlen));
                return true;
            }
        }
        return false;
    } finally {
        lock.unlock();
    }
}

// 2.默认package访问，删除某一位置范围的元素
void removeRange(int fromIndex, int toIndex) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 边界检查
        if (fromIndex < 0 || toIndex > len || toIndex < fromIndex)
            throw new IndexOutOfBoundsException();
        int newlen = len - (toIndex - fromIndex);
        int numMoved = len - toIndex;
        if (numMoved == 0) //不需要移动元素
            setArray(Arrays.copyOf(elements, newlen));
        else {
            Object[] newElements = new Object[newlen];
            System.arraycopy(elements, 0, newElements, 0, fromIndex);
            System.arraycopy(elements, toIndex, newElements,
                             fromIndex, numMoved);
            setArray(newElements);
        }
    } finally {
        lock.unlock();
    }
}

// 3.删除符合条件的元素
public boolean removeIf(Predicate<? super E> filter) {
    if (filter == null) throw new NullPointerException(); 
    final ReentrantLock lock = this.lock;
    lock.lock(); //竞争锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (len != 0) {
            int newlen = 0;
            // temp保存保留下的元素
            Object[] temp = new Object[len];
            for (int i = 0; i < len; ++i) {
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                if (!filter.test(e))
                    temp[newlen++] = e;
            }
            if (newlen != len) { //有元素被删除，COW
                setArray(Arrays.copyOf(temp, newlen));
                return true;
            }
        }
        // 链表中不含有符合传入条件的元素，返回false.
        return false;
    } finally {
        lock.unlock();
    }
}
```
##### 3.4 get & set
get和set方法比较简单
```java
public E get(int index) {
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) { //COW，新数组替换
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements); 
        } else { //没有发生改变，==即内存地址都一样
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
##### 3.5 clone方法
CopyOnWriteArrayList的克隆方法只是一个浅拷贝，调用父类中定义的clone方法.
```java
public Object clone() {
    try {
        @SuppressWarnings("unchecked")
        CopyOnWriteArrayList<E> clone =
            (CopyOnWriteArrayList<E>) super.clone(); 
        clone.resetLock(); //反序列化时重置锁
        return clone;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError();
    }
}

// 重置锁，借助Unsafe指针
private void resetLock() {
    UNSAFE.putObjectVolatile(this, lockOffset, new ReentrantLock());
}

```
##### 3.6 writeObject方法 


```java
// 反序列化调用方法
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    // 重置锁，因为lock字段时被transient修饰
    resetLock();
    // Read in array length and allocate array
    int len = s.readInt();
    Object[] elements = new Object[len];

    // Read in all elements in the proper order.
    for (int i = 0; i < len; i++)
        elements[i] = s.readObject();
    setArray(elements);
}

// 序列化调用
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    s.defaultWriteObject(); //先调用默认序列化方法

    Object[] elements = getArray();
    // 写入底层数组的大小
    s.writeInt(elements.length);

    // 向流中有序写入所有元素
    for (Object element : elements)
        s.writeObject(element);
}
```

##### 3.7 sort方法
sort方法同样采用COW机制.
```java
public void sort(Comparator<? super E> c) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        Object[] newElements = Arrays.copyOf(elements, elements.length);
        @SuppressWarnings("unchecked") E[] es = (E[])newElements;
        Arrays.sort(es, c);
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
```

#### 4.迭代器
CopyOnWriteArrayList的迭代器不支持对链表的修改，且在迭代过程中对其它线程对链表的修改并不能及时感知到，是一种若一致性的迭代器
```java
static final class COWIterator<E> implements ListIterator<E> {
    // 保存底层数组的快照，对其它线程的修改感觉不到，弱一致性
    private final Object[] snapshot;
    // 记录已经迭代的位置
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }
    // 是否还有未迭代的元素
    public boolean hasNext() {
        return cursor < snapshot.length;
    }
    // 是否有迭代过的元素
    public boolean hasPrevious() {
        return cursor > 0;
    }

    // 向前迭代
    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }

    // 向后迭代
    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }
    // 下一个向前迭代的元素
    public int nextIndex() {
        return cursor;
    }
    public int previousIndex() {
        return cursor-1;
    }

    /* 迭代过程中的增删改都是不支持的，调用会抛出异常 */
    public void remove() {
        throw new UnsupportedOperationException();
    }
    public void set(E e) {
        throw new UnsupportedOperationException();
    }
    public void add(E e) {
        throw new UnsupportedOperationException();
    }

    @Override
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        Object[] elements = snapshot;
        final int size = elements.length;
        for (int i = cursor; i < size; i++) {
            @SuppressWarnings("unchecked") E e = (E) elements[i];
            action.accept(e);
        }
        cursor = size;
    }
}
```

#### 5.总结
+ CopyOnWriteArrayList的增删改，以及排序等操作都使用了写时复制策略，而这些操作中出现较多的获取-拷贝-写入(替换)三个步骤并不是原子性的，因此使用了重入锁进行保障。

+ 数据一致性：CopyOnWriteArrayList只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。

+ 内存占用：因为写时复制，在写时会有两份内存占用，新旧底层数组。若存放的是大对象，会导致较为频繁的年轻代GC和老年代GC。 

+ 因为写时复制的内存占用以及锁竞争的原因，CopyOnWriteArrayList适合读多写少的场景。

+ 此外，CopyOnWriteArrayList提供了弱一致性的迭代器，保证在获取迭代器后，其他线程对list的修改当前线程不可见，迭代器遍历时候的数组是获取迭代器时候的一个快照。