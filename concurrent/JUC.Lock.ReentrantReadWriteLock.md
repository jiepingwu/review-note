### ReentrantReadWriteLock

前面我们学习了 互斥锁的实现 ReentrantLock 和 共享锁实现 Semaphore。

建议好好理解这两个实现，再来看 ReentrantReadWriteLock。

ReentrantReadWriteLock 既有 互斥锁 又有 共享锁，具体来说，ReadLock 共享，WriteLock互斥。

同样地，ReentrantReadWriteLock 也是基于 AQS 实现。



#### ReentrantReadWriteLock

直译过来就是 **可重入读写锁**。

常见场景是：对于 读多写少 的业务场景。比如应用缓存。

**一个线程将数据写入缓存，其它线程可以直接读取缓存中的数据，提高数据查询效率。**

对于可以共享读的业务场景，使用纯互斥锁就很低效，所以我们可以使用读写锁，读共享，写互斥，也就是说，在同一时刻允许多个线程进行读操作获取读锁，而只能有一个线程获取写锁。

读写锁模型3条规定：

1. 允许多个线程同时读共享变量。
2. 只允许一个线程写共享变量。
3. 如果写线程正在执行写操作，则禁止其它线程读共享变量。

ReadWriteLock 是一个接口，内部只有两个方法：

```java
public interface ReadWriteLock {
    // 返回用于读的锁
    Lock readLock();

    // 返回用于写的锁
    Lock writeLock();
}
```



##### ReentrantReadWriteLock 的类结构

![img](https://rgyb.sunluomeng.top/20200621160819.png)

很相似吧，可以发现 ReentrantReadWriteLock 的基本特性：

![img](https://rgyb.sunluomeng.top/20200621161359.png)

注意这里的 **锁降级**（后面详细分析）：**遵循 1. 获取写锁 2. 获取读锁 3. 释放写锁 的顺序。也就是锁，写锁需要先获取读锁才能降级为读锁。**

前面说过，Lock 和 AQS 以【聚合】的方式嵌入，这里的读写锁两种锁，也就存在两种聚合。

```java
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
```

![img](https://rgyb.sunluomeng.top/20200621155924.png)

##### ReadWriteLock 示例

这里给出一个模拟使用缓存的示例。(缓存中数据对于读共享，对于写互斥，并且写对于读互斥。)

```java
public class ReentrantReadWriteLockCache {
    static Map<String, Object> cache = new HashMap(String, Object);
    static ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    static Lock r1 = readWriteLock.readLock(); // read lock
    static Lock w1 = readWriteLock.writeLock(); // write lock
    
    public static final Object get(String key) {
        r1.lock(); // read lock
        try {
            return cache.get(key);
        } finally {
            r1.unlock();
        }
    }
    
    public static final Object put(String key, Object value) {
        w1.lock();
        try {
            return cache.put(key, value);
        } finally {
            w1.unlock();
        }
    }
}
```

##### 读写状态如何实现？

我们知道，AQS使用一个 volatile int state 变量来实现 🔒 的语义，那么 一个 int 变量 如何提供两种(读，写)锁 的同步状态呢？

如果你看过 ThreadPool 的 ThreadPoolExecutor 实现，你会发现其中 ThreadPool 的状态 和 WorkerCount 数量也是维护在一个 int 变量中的。

**同样的，这里也是使用 高低位 分别代表不同的含义来实现的，具体地：**

- **高16位代表 写状态。**

- **低16位代表 读状态。**

我们也就知道一个"奇怪的知识"：读写锁可重入的次数(2^16 - 1) 是 ReentrantLock这种互斥锁(2^32 - 1)的 0.5次方。

![img](https://rgyb.sunluomeng.top/20200621133150.png)

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;


        static int sharedCount(int c) { 
          return c >>> SHARED_SHIFT; 
        }

        static int exclusiveCount(int c) { 
          return c & EXCLUSIVE_MASK; 
        }
}
```

---

#### 源码分析

##### writeLock 分析

由于写锁是互斥的，所以重写AQS中的 tryAcquire 方法。

```java
protected final boolean tryAcquire(int acquires) {        
    Thread current = Thread.currentThread();
  	// 获取 state 整体的值
    int c = getState();
    // 获取写状态的值
    int w = exclusiveCount(c);
    if (c != 0) {
        // w=0: 根据推理二，整体状态不等于零，写状态等于零，所以，读状态大于0，即存在读锁
      	// 或者当前线程不是已获取写锁的线程
      	// 二者之一条件成真，则获取写状态失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 根据推理一第 1 条，更新写状态值
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||	// writerShouldBlock() 是在 公平锁 和 非公平锁 中的不同实现，判断是否有前驱节点。
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

![img](https://rgyb.sunluomeng.top/20200621144015.png)

##### readLock 分析

读锁共享，重写 AQS 中的 tryAcquireShared 方法。

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
  	// 写状态不等于0，并且锁的持有者不是当前线程，根据约定 3，则获取读锁失败
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
  	// 获取读状态值
    int r = sharedCount(c);
  	// 这个地方有点不一样，我们单独说明
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;	// 此处使用 ThreadLocal 记录本地线程读锁状态计算
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
  	// 如果获取读锁失败则进入自旋获取
    return fullTryAcquireShared(current);
}
```

**其中 readerShouldBlock()判断是否阻塞时有点不同。**

在公平锁的实现上，readerShouldBlock 和 writerShouldBlock 都是判断是否存在前驱节点，而在非公平锁实现上：

readerShouldBlock()是向下面这样的：

```java
final boolean readerShouldBlock() {
	return apparentlyFirstQueuedIsExclusive();
}

final boolean apparentlyFirstQueuedIsExclusive() {
  Node h, s;
  return (h = head) != null &&
    // 等待队列头节点的下一个节点
    (s = h.next)  != null &&
    // 如果是排他式的节点
    !s.isShared()         &&
    s.thread != null;
}
```

**也就是说，如果 请求读锁的线程 发现 同步队列的head节点的下一个节点为 排他式节点，那么 说明 有一个线程在等待 获取 写锁。**

**此时，就将请求读锁的线程阻塞。**

这种机制的实现是为了：**提高写锁的获取优先级，防止写锁饥饿**。毕竟 读多写少。



##### ThreadLocal 记录本地 线程同步状态计数

由于读锁是共享的，为了记录每个线程 本身持有的锁状态计数，这里使用 **ThreadLocal** 来记录。

key 为 ThreadLocal threadLocalHoldCounter 对象，value为 HoldCounter，其中持有 状态计数count 和 对应的线程 id。

![img](https://rgyb.sunluomeng.top/20200621165753.png)



##### 读写锁的升级和降级

读锁是可以被多线程共享的，而写锁是线程独占的，也就是说 写锁 的并发限制比 读锁 高。

所以，从读锁可以升级为 写锁，从 写锁可以降级为 读锁。

![img](https://rgyb.sunluomeng.top/20200621173215.png)

先完善一下开头的ReentrantReadWriteLock 的例子。

```java
public static final Object get(String key) {
    Object obj = null;
    r1.lock();
    try {
        // 获取缓存中的值
        obj = cache.get(key);
    } finally {
        r1.unlock();
    }
    // 如果缓存命中，直接返回
    if (obj != null) {
        return obj;
    }
    
    // 缓存没有命中，通过写锁查询DB，并将其写入缓存中
    w1.lock();
    try {
        // 再次尝试获取缓存中的值
        obj = cache.get(key);
        if (obj == null) {
            obj = getDataFromDB(key); // 从DB中查询数据
            cache.put(key, obj);
        }
    } finally {
        w1.unlock();
    }
    return obj;
}
```

为什么在写锁里面，还要再次获取缓存中的值呢？

这里的 double check 的原因是：**可能多线程同时执行到 get方法，其中只有一个线程获取锁执行get，这个时候cache已经更新了。也就是说，可以获取到锁的时候，其它线程已经更新了cache。**

**如下图：线程A，B，C同时尝试获取写锁w1，此时只有A线程获取写锁，B,C线程阻塞，等到B，C线程获取到锁的时候，缓存cache已经被线程A更新了。也就不需要再次getDataFromDB，然后更新缓存了，这里可能可以减少一次DB操作。**

![image-20210601191425580](..\references-figures\image-20210601191425580.png)

###### 锁升级

那我们能不能这样，直接在获取读锁时，如果缓存没有命中，直接获取写锁然后从DB中查询然后更新缓存？

不行，因为 **获取一个写锁需要先释放全部的读锁！**

如果两个获取了读锁的线程同时尝试获取写锁，且都不释放读锁时，就会发生死锁。

所以，锁的直接升级是不允许的（即持有读锁状态下不允许获取写锁，需要先释放全部读锁）



###### 锁降级

锁降级可以吗？

下面是 Oracle官方锁降级示例：

```java
class CachedData {
    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
    
    void processCachedData() {
        rw.readLock().lock(); // read lock
        if (!cacheValid) {
            // 必须在获取写锁之前释放读锁，因为锁升级不被允许
			rw.readLock().unlock();
            rw.writeLock().lock();
            try {
                // check again，因为其它线程可能已经更新缓存
                if (!cacheValid) {
                    data = ...;
                    cacheValid = true;
                }
                // 释放写锁前，降级为读锁
                rw.readLock().lock();
            } finally {
                rw.writeLock().unlock();
            }
        }
        try {
            use(data);
        } finally {
            rw.readLock().unlock();
        }
    }
}
```

1. 首先获取读锁，如果cache不可用，则释放读锁。
2. 获取写锁。
3. 更新cache之前，再检查cache是否可用，因为其它线程可以已经更新cache。然后更新cache，将更新标志位cacheValied置为true。
4. 然后在 **释放写锁前获取读锁**
5. 此时cache可用，处理cache中数据，然后释放读锁。

这是整个锁降级的过程，关键就是 **释放写锁前要先获取读锁，目的是为了保证数据的可见性。**

为什么这样做能保证数据的可见性呢？或者说我们不这样，在释放写锁前如果不获取读锁会怎么样？

​	如果线程A在更新了 cache 之后，没有获取读锁，直接释放了写锁，假设此时另一个线程B获取了写锁并修改了数据，那么线程A将无法感知到数据被修改，但是线程A还应用了cache数据，就可能导致数据错误。

​	而如果我们遵循锁降级，线程A在更新了cache之后，先获取读锁，再释放写锁，那么线程B获取写锁时将被阻塞，知道线程A处理完数据后释放读锁，线程B才能获取到写锁修改数据，保证数据的可见性和正确性。



那么：**读写锁一定要进行锁降级吗？**

​	这里其实不一定，我们通过上面过程发现，锁降级其实是保证了 线程中的视图操作，即 我们如果想 在一个线程中进行一个按照一致性视图数据的操作，就需要锁降级，防止其它线程在当前线程操作数据时修改了数据。相反，如果我们不需要 保证一致性视图，就不需要使用锁降级，也就是说允许当前线程实时地读到其它线程的修改。

