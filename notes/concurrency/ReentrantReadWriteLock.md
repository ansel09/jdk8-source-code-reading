### *ReentrantReadWriteLock*可重入读写锁
---
- ReentrantReadWriteLock实现了ReadWriteLock接口。
- 读共享，写互斥，对于共享资源读多写少的情况，其比单纯的ReentrantLock等独占锁更合适。

#### 1. ReadWriteLock读写锁接口
```java
// 读写锁接口
public interface ReadWriteLock {
    Lock readLock();  //Returns the lock used for reading.
    Lock writeLock(); //Returns the lock used for writing.
}
```
#### 2. 