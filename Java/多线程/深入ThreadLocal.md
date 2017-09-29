```java
public class ThreadLocal<T> {
    public T get() {
        Thread t = Thread.currentThread();
        //从当前的Thread对象中取出ThreadLocalMap成员，key是ThreadLocal，value是set的值。
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        //从当前的Thread对象中取出ThreadLocalMap成员，key是ThreadLocal，value是set的值。
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

}

//threadLocals变量存放在Thread对象中
public class Thread{
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

    ...
}
```

每个ThreadLocal对象内部都没有保存Value对象，而是将Value存到调用线程所属的ThreadLocalMap里面，ThreadLocalMap的key为ThreadLocal，value就是线程本地变量。ThreadLocalMap对ThreadLocal的引用是弱引用。

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```



为什么不构造一个Map，以Thread对象为key呢？原因有二：
1. 集合类强引用Thread对象容易造成内存泄漏。
2. 需要处理线程同步问题，加锁会有性能开销。


ThreadLocalMap采用的开放地址法来解决key的冲突问题，而不是HashMap的链地址法。
为什么采用开发地址法呢？我的理解是：一个Thread对象的线程私有变量一般不会太多，因此用开放地址法来解决冲突可以提高hash map的空间利用率。

总的来说，对于多线程资源共享的问题，同步机制采用了“**以时间换空间**”的方式，而ThreadLocal采用了“**以空间换时间**”的方式。前者仅提供一份变量，让不同的线程排队访问；而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。



http://www.sczyh30.com/posts/Java/java-concurrent-threadlocal/