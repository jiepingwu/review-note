#### ThreadLocal

##### ThreadLocal 的用处

​	ThreadLocal用于保存线程隔离的变量，也就是说每个线程持有自己独占的变量。可以防止线程访问到其它用户的数据。

##### ThreadLocal的原理

​	每个线程 Thread 都持有一个 ThreadLocalMap 对象，该ThreadLocalMap 是一个Entry[]数组，Entry为一组键值对，key保存了 ThreadLocal对象，value保存对应保存的值。

```java
// 调用方式如下
ThreadLocal<String> local = new ThreadLocal<>();
local.set("hello thread local");

ThreadLocal<String> local_val = local.get();
```

当调用 ThreadLocal local.set("hello thread local")时，先拿到 Thread.currentThread.getThreadLocalMap() ThreadLocalMap成员变量，通过threadLocalMap.set(threadLocal, value)；set进去对应ThreadLocal的value。

*这里为什么key 不是 Thread，而是ThreadLocal？*

因为如果是Thread，那么一个线程只能保存一个本地变量，每次set时，就会覆盖原有的值。使用threadLocal作为key，可以保存多个threadLocal变量作为key的变量。



ThreadLocalMap 的key是一个 弱引用，原因在于：为了防止 ThreadLocal 被引用而导致的内存泄露。

因为Thread持有了 ThreadLocalMap，而ThreadLocalMap的key为ThreadLocal，所以如果key为强引用，则 key的生命周期和 Thread 一样，只有线程Thread结束释放资源时，ThreadLocalMap的key ThreadLocal才可以被回收，这就回产生内存泄漏。所以，它key ThreadLocal是一个弱引用，只要当 GC线程看到了这个引用，那么就会被回收。

另外，ThreadLocalMap的value也可能产生内存泄漏，因为当key被回收变为null后，value并没有回收。所以jdk源码中，会在 每次get、set时，清除key为null的entry对应的value为null。并且，每次手动进行remove也可以清理key为null的value。



