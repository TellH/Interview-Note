- **1.**LruCache 是通过 LinkedHashMap 构造方法的第三个参数的 `accessOrder=true` 实现了 `LinkedHashMap` 的数据排序**基于访问顺序** （最近访问的数据会在链表尾部），在容量溢出的时候，将链表头部的数据移除。从而，实现了 LRU 数据缓存机制。
- **2.**LruCache 在内部的get、put、remove包括 trimToSize 都是安全的（因为都上锁了）。
- **3.**LruCache 自身并没有释放内存，将 LinkedHashMap 的数据移除了，如果数据还在别的地方被引用了，还是有泄漏问题，还需要手动释放内存。
- **4.**覆写 `entryRemoved` 方法能知道 LruCache 数据移除是是否发生了冲突，也可以去手动释放资源。
- **5.**`maxSize` 和 `sizeOf(K key, V value)` 方法的覆写息息相关，必须相同单位。（ 比如 maxSize 是7MB，自定义的 sizeOf 计算每个数据大小的时候必须能算出与MB之间有联系的单位 ）


每次调用LinkedHashMap的get方法时，如果`accessOrder=true` ，那么它会调用`makeTail`方法，将entry移到链表的尾部。



https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/LruCache%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md





**LinkedHashMap** = HashMap + 双向链表

```java
private static class Entry<K,V> extends HashMap.Entry<K,V> {
    // These fields comprise the doubly linked list used for iteration.
    Entry<K,V> before, after;

Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
        super(hash, key, value, next);
    }
    ...
}
```

Entry拥有的属性：

- **K key**
- **V value**
- **Entry<K, V> next**
- **int hash**
- **Entry<K, V> before**
- **Entry<K, V> after**