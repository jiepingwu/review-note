### AQS

#### 写在前面

AQS全名叫做AbstractQueueSynchronizer，是JUC中的一个抽象类。提供了同步器的抽象实现，可以使用来实现锁的语义。

都说看JUC就必须先看AQS，下面我们就来看看这个AQS。



#### Java SDK为什么还要设计Lock？

​	在1.5之后，Doug Lea大师就重新造了一个轮子 Lock。既然有了 Synchronized 为什么还需要 Lock呢？

Synchronized内置锁有什么问题呢？

​	使用Synchronized，如果线程申请不到锁资源就会一直处于阻塞状态，我们怎样也改变不了它的状态。这是内置锁最大的问题，我们不能中断地获取锁，也不能使其中断，不能抛出异常，不能 记时等待获取锁。

​	于是，需要Doug Lea 重造轮子 Lock。

#### Lock 显示锁

新轮子Lock怎么解决上面的问题的呢？

**特性	**						**描述** 																								**API**

能响应中断				如果不能自己释放，需要通过中断信号跳出某种状态。		lockInterruptbly()

非阻塞地获取锁		尝试获取，获取失败不会阻塞，直接返回								tryLock()

超时获取					获取超时后，不进入阻塞，直接返回										tryLock(long time, TimeUnit t)

**Lock 范式**

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // do something...
} finally {
    lock.unLock();
}
```

1. finally中释放锁。(保证一定最终会释放锁。)
2. try外部获取锁，并且获取锁应该是方法的第一行。（try外部获取锁是因为：1.如果没有获取到锁就抛出异常，那么最终释放锁肯定就有问题。2.如果在获取锁时抛出了异常，也就是当前线程没有获取到锁，但执行到finally代码时，如果其它线程获取到了锁，就会被这里的unLock无故释放掉。

**Lock 是怎么实现锁的语义的？**

​	如果和 Synchronized 对比的话，lock方法就相当于 moniterenter ， unlock方法就相当于 moniterexit。

​	Lock 通过内部维护一个 volatile 变量 state，通过CAS对这个变量进行读写，如果CAS更改成功，表示获取到锁，如果CAS失败，那么将当前线程unpark挂起。

![img](https://rgyb.sunluomeng.top/20200523141526.png)

发现Lock中没有这个state，Lock只是一个接口，Lock的实现类基本上都是【聚合】了一个 【队列同步器】的实现类 来完成线程的访问控制的。

也就是 AQS 了。



#### AQS 队列同步器

AbstractQueuedSynchronizer，AQS。

![img](https://rgyb.sunluomeng.top/20200517201817.png)

看上图，**ReentrantLock，ReentrantReadWriteLock，Semaphore，CountDownLatch，ThreadPoolExecutor 中都实现了AQS。**

AQS是一个抽象类，并且使用 【模板模式】来实现，使用者可以重写指定方法，然后调用模板方法，模板方法中会调用使用者重写的方法。

##### AQS中可以重写的方法

记忆上，分为独占式和共享式。

**独占式获取 tryAcquire(int arg) 和 释放 tryRelease(int arg)**，

**共享式获取 tryAcquireShared(int arg) 和 释放 tryReleaseShared(int arg)**，

获取**当前同步器是否为独占模式 isHeldExclusively()。** 

![img](https://rgyb.sunluomeng.top/20200523160830.png)

​	一目了然，分为独占式和共享式获取和释放锁，以及 当前同步器是否为独占模式的方法。这里方法不是 abstract 的原因是：一个锁要么是独占式的，要么是共享式的，为了避免强制需要重写不相干的方法，就没有使用abstract修饰，但是方法会抛出 UnsupportedOperationException。

##### 同步状态

​	volatile state，在不同的AQS实现中，会有不同的锁语义。使用下面的方法来获取或者设置同步状态。

![img](https://rgyb.sunluomeng.top/20200523160906.png)

##### AQS中的模板方法

记忆上，也是可以分为独占式和共享式。

两种都有这4个API：除了acquire获取和release释放同步状态，还有 可中断的请求锁，带超时的请求锁。

独占式：acquire(int arg)，release(int arg)，acquireInterruptibly(int arg)，tryAcquireNanos(int arg, long nanoTimeout)。

共享式：也是一样一套，API名字+shared。

##### ![img](https://rgyb.sunluomeng.top/20200523195957.png)

整体来看是这样：**如果我们需要实现一个自定义的锁。**

我们需要

1. 实现Lock接口，实现类： **MyMutex** implemets Lock。
2. MyMutex中聚合一个 自定义的AQS继承自抽象类AQS：  **MySync** extends AQS。
3.  MySync中实现需要我们重写的方法：**tryAcquire(tryAcquireShared)、tryRelease(tryReleaseShared)、isHeldExclusively**。
4. MySync中重写的方法会调用 AQS中的【模板方法】- **acquire、release、acquireInterruptibly、tryAcquireNanos等**。 

![image-20210530212145767](..\references-figures\image-20210530212145767.png)

### AQS的实现分析

#### Node 节点

AQS内部维护了一个队列，用于管理同步状态。

- 当线程获取同步状态失败时，就将当前线程和等待状态等信息构造成一个Node节点，将其加入到 同步队列 的尾部，将其挂起。
- 当同步状态释放的时候，会唤醒同步队列中的"首节点"的线程获取同步状态。

![img](https://rgyb.sunluomeng.top/20200524183916.png)

![img](https://rgyb.sunluomeng.top/20200524184014.png)![img](https://rgyb.sunluomeng.top/20200525072245.png)



#### 独占式锁

##### 独占式获取同步状态

从lock.lock()开始

```java
public void lock() {
	// 阻塞式的获取锁，调用同步器模版方法，获取同步状态
	sync.acquire(1);
}
```

进入AQS的模板方法 acquire()

```java
public final void acquire(int arg) {
  // 调用自定义同步器重写的 tryAcquire 方法
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```

首先，尝试非阻塞的获取同步状态。

如果获取失败(tryAcquire返回false)，则调用 **addWaiter方法构造Node节点(Node.EXCLUSIVE独占式) 并安全的CAS加入到同步队列的尾部。**

```java
private Node addWaiter(Node mode) {
  	// 构造Node节点，包含当前线程信息以及节点模式【独占/共享】
    Node node = new Node(Thread.currentThread(), mode);
  	// 新建变量 pred 将指针指向tail指向的节点
    Node pred = tail;
  	// 如果尾节点不为空
    if (pred != null) {
      	// 新加入的节点前驱节点指向尾节点
        node.prev = pred;

      	// 因为如果多个线程同时获取同步状态失败都会执行这段代码
        // 所以，通过 CAS 方式确保安全的设置当前节点为最新的尾节点
        if (compareAndSetTail(pred, node)) {
          	// 曾经的尾节点的后继节点指向当前节点
            pred.next = node;
          	// 返回新构建的节点
            return node;
        }
    }
  	// 尾节点为空，说明当前节点是第一个被加入到同步队列中的节点
  	// 需要一个入队操作
    enq(node);
    return node;
}

private Node enq(final Node node) {
  	// 通过“死循环”确保节点被正确添加，最终将其设置为尾节点之后才会返回，这里使用 CAS 的理由和上面一样
    for (;;) {
        Node t = tail;
      	// 第一次循环，如果尾节点为 null
        if (t == null) { // Must initialize
          	// 构建一个哨兵节点，并将头部指针指向它
            if (compareAndSetHead(new Node()))
              	// 尾部指针同样指向哨兵节点
                tail = head;
        } else {
          	// 第二次循环，将新节点的前驱节点指向t
            node.prev = t;
          	// 将新节点加入到队列尾节点
            if (compareAndSetTail(t, node)) {
              	// 前驱节点的后继节点指向当前新节点，完成双向队列
                t.next = node;
                return t;
            }
        }
    }
}
```

acquireQueued()方法

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
      	// "死循环"，尝试获取锁，或者挂起
        for (;;) {
          	// 获取当前节点的前驱节点
            final Node p = node.predecessor();
          	// 只有当前节点的前驱节点是头节点，才会尝试获取锁
          	// 看到这你应该理解添加哨兵节点的含义了吧
            if (p == head && tryAcquire(arg)) {
              	// 获取同步状态成功，将自己设置为头
                setHead(node);
              	// 将哨兵节点的后继节点置为空，方便GC
                p.next = null; // help GC
                failed = false;
              	// 返回中断标识
                return interrupted;
            }
          	// 当前节点的前驱节点不是头节点
          	//【或者】当前节点的前驱节点是头节点但获取同步状态失败
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

上面 获取同步状态成功就返回，那 获取同步状态失败后会一直陷入到 "死循环" 中浪费资源吗？



显然不是，**shouldParkAfterFailedAcquire(p, node) 和 parkAndCheckInterrupt() 方法会将 获取同步状态失败的线程 挂起。**

shouldParkAfterFailedAcquire方法

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  	// 获取前驱节点的状态
    int ws = pred.waitStatus;
  	// 如果是 SIGNAL 状态，即等待被占用的资源释放，直接返回 true
  	// 准备继续调用 parkAndCheckInterrupt 方法
    if (ws == Node.SIGNAL)
        return true;
  	// ws 大于0说明是CANCELLED状态，
    if (ws > 0) {
        // 循环判断前驱节点的前驱节点是否也为CANCELLED状态，忽略该状态的节点，重新连接队列
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
      	// 将当前节点的前驱节点设置为设置为 SIGNAL 状态，用于后续唤醒操作
      	// 程序第一次执行到这返回为false，还会进行外层第二次循环，最终从代码第7行返回
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

该方法将获取同步状态失败的线程节点，放置在同步队列中的 **合适的位置。**合适的位置 指 在 SIGNAL节点的后面。SIGNAL节点被唤醒后，会自动通知后继节点尝试获取同步状态。

在这个方法中，如果 前驱节点的waitStatus是 SIGNAL 状态，就直接返回 true，程序会向下继续执行 **parkAndCheckInterrupt()方法，用于将 当前线程 挂起park。**



parkAndCheckInterrupt方法

```java
private final boolean parkAndCheckInterrupt() {
  	// 线程挂起，程序不会继续向下执行
    LockSupport.park(this);
  	// 根据 park 方法 API描述，程序在下述三种情况会继续向下执行
  	// 	1. 被 unpark 
  	// 	2. 被中断(interrupt)
  	// 	3. 其他不合逻辑的返回才会继续向下执行
  	
  	// 因上述三种情况程序执行至此，返回当前线程的中断状态，并清空中断状态
  	// 如果由于被中断，该方法会返回 true
    return Thread.interrupted();
}
```

该方法通过 LockSupport.park(this) 将当前线程挂起(所以使用lock方法阻塞的线程状态为 BLOCKING)。

线程被unpark唤醒后，继续执行 acquireQueued方法里的循环，如果获取同步状态成功，返回 interrupted = true 的结果。

程序继续向调用栈上层返回，最终回到AQS中的模板方法 acquire。

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```

**这里为什么要有自我中断呢？**

中断标志被 Thread.interrupted()清空了，所以自己要重新设立一下中断标识。

**acquireQueued中的finally代码，什么时候才会执行 if(failed)的代码？**

```java
if (failed)
    cancelAcquire(node);
```

正常情况下，如果跳出循环，failed的值为false，如果不跳出循环就不能执行到这里，所以这里是发生异常时才会执行。

发生什么异常呢？1. node.processor()抛出的NullPointerException。2. tryAcquire()方法抛出的异常，比如 ReentrantLock中 如果 nextc = c + acquires < 0，则会抛出 Error("Maximum lock count exceeded")。



另外，上面 shouldParkAfterFailedAcquire方法中还对 CANCELLED 节点状态进行了判断，那么什么时候会生成 CANCELLED 状态的 node呢？

cancelAcquire方法

```java
private void cancelAcquire(Node node) {
       // 忽略无效节点
       if (node == null)
           return;
			// 将关联的线程信息清空
       node.thread = null;

       // 跳过同样是取消状态的前驱节点
       Node pred = node.prev;
       while (pred.waitStatus > 0)
           node.prev = pred = pred.prev;

       // 跳出上面循环后找到前驱有效节点，并获取该有效节点的后继节点
       Node predNext = pred.next;

       // 将当前节点的状态置为 CANCELLED
       node.waitStatus = Node.CANCELLED;

       // 如果当前节点处在尾节点，直接从队列中删除自己就好
       if (node == tail && compareAndSetTail(node, pred)) {
           compareAndSetNext(pred, predNext, null);
       } else {
           int ws;
         	// 1. 如果当前节点的有效前驱节点不是头节点，也就是说当前节点不是头节点的后继节点
           if (pred != head &&
               // 2. 判断当前节点有效前驱节点的状态是否为 SIGNAL
               ((ws = pred.waitStatus) == Node.SIGNAL ||
                // 3. 如果不是，尝试将前驱节点的状态置为 SIGNAL
                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
               // 判断当前节点有效前驱节点的线程信息是否为空
               pred.thread != null) {
             	// 上述条件满足
               Node next = node.next;
             	// 将当前节点有效前驱节点的后继节点指针指向当前节点的后继节点
               if (next != null && next.waitStatus <= 0)
                   compareAndSetNext(pred, predNext, next);
           } else {
             	// 如果当前节点的前驱节点是头节点，或者上述其他条件不满足，就唤醒当前节点的后继节点
               unparkSuccessor(node);
           }
					
           node.next = node; // help GC
       }
```

这个 cancelAcquire()方法的目的是：**从等待队列中移除 CANCELLED 的节点，并重新拼接整个队列。**

至此，获取锁的过程就结束了，下面是整个过程：

![img](https://rgyb.sunluomeng.top/20200527112235.png)

总结来说：

- lock获取同步状态时，以**CAS的方式 修改 volatile state变量的值**，如果成功表示获取同步状态**成功**。
- 如果获取同步状态**失败**，则**生成当前线程和节点状态 封装成 node节点**，将节点以 **CAS+自旋 的方式插入到 同步队列的尾部**。
- 在同步队列中 **自旋** 判断 **当前节点是应该被唤醒尝试获取锁** 还是 **被挂起**，被挂起的时候会将挂起的节点 放置到 SIGNALLED 节点后面，然后LockSupport.park()将当前线程挂起。另外，只有node节点的前驱节点是head节点，node节点才会被唤醒去请求锁。



##### 独占式释放同步状态

从unlock()方法开始。

```java
public void unlock() {
	// 释放锁
	sync.release(1);
}
```

调用AQS模板方法 release()。

```java
public final boolean release(int arg) {
  	// 调用自定义同步器重写的 tryRelease 方法尝试释放同步状态
    if (tryRelease(arg)) {
      	// 释放成功，获取头节点
        Node h = head;
      	// 存在头节点，并且waitStatus不是初始状态
      	// 通过获取的过程我们已经分析了，在获取的过程中会将 waitStatus的值从初始状态更新成 SIGNAL 状态
        if (h != null && h.waitStatus != 0)
          	// 解除后继线程挂起状态
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

tryRelease() 释放同步状态成功后，调用 unparkSuccessor(head) 唤醒head节点的后继节点。

unparkSuccessor(Node h)方法

```java
private void unparkSuccessor(Node node) {      
  	// 获取头节点的waitStatus
    int ws = node.waitStatus;
    if (ws < 0)
      	// 清空头节点的waitStatus值，即置为0
        compareAndSetWaitStatus(node, ws, 0);
  
  	// 获取头节点的后继节点
    Node s = node.next;
  	// 判断当前节点的后继节点是否是取消状态，如果是，需要移除，重新连接队列
    if (s == null || s.waitStatus > 0) {
        s = null;
      	// 从尾节点向前查找，找到队列第一个waitStatus状态小于0的节点
        for (Node t = tail; t != null && t != node; t = t.prev)		// !!! 一个细节：唤醒时从尾部向前查找
          	// 如果是独占式，这里小于0，其实就是 SIGNAL
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
      	// 解除线程挂起状态
        LockSupport.unpark(s.thread);
}
```

同步状态释放后，之前获取同步状态被park的线程会被唤醒，继续轮询尝试获取锁。



##### 独占式响应中断获取同步状态

从lock.lockInterruptibly()方法开始。

```java
public void lockInterruptibly() throws InterruptedException {
	// 调用同步器模版方法可中断式获取同步状态
	sync.acquireInterruptibly(1);
}
```

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
  	// 尝试非阻塞式获取同步状态失败，如果没有获取到同步状态，执行代码7行
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
              	// 获取中断信号后，不再返回 interrupted = true 的值，而是直接抛出 InterruptedException 
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

中断抛出 InterruptedException 异常。



##### 独占式超时限制获取同步状态

lock.tryLock(time, timeunit)方法

```java
public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
	// 调用同步器模版方法，可响应中断和超时时间限制
	return sync.tryAcquireNanos(1, unit.toNanos(time));
}
```

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

方法标记上有 `throws InterruptedException` 说明该方法也是可以响应中断的，所以你可以理解超时限制是 `acquireInterruptibly` 方法的加强版，具有超时和非阻塞控制的双保险。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
  	// 超时时间内，为获取到同步状态，直接返回false
    if (nanosTimeout <= 0L)
        return false;
  	// 计算超时截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
  	// 以独占方式加入到同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
          	// 计算新的超时时间
            nanosTimeout = deadline - System.nanoTime();
          	// 如果超时，直接返回 false
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
            		// 判断是最新超时时间是否大于阈值 1000    
                nanosTimeout > spinForTimeoutThreshold)
              	// 挂起线程 nanosTimeout 长时间，时间到，自动返回
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

上面的方法好理解，其中判断超时时间是否大于1000的原因在于：1000ns已经很短了，没有必要再进行挂起和唤醒操作，直接进行下一次循环就行。



AQS 相当于就结束了，这里只聊了 独占式获取同步状态的方法。下面还有 共享式锁。

---

#### 共享式锁

共享式 和 独占式 的区别在于：同一时刻是否有多个线程可以获取到同步状态？(是则共享，否则独占)

对于实现来说，同步状态state维护在AQS中，对于 独占式：只有在state=0时才能获取同步状态成功。对于 共享式：不管state是否为0，都可以获取同步状态成功，即state++。

##### 共享式获取同步状态

重放两个关键图：

**AQS中可以重写的方法**

![img](https://rgyb.sunluomeng.top/20200523160830.png)

**AQS中提供的模板方法**

![img](https://rgyb.sunluomeng.top/20200523195957.png)

一样地，你会发现和独占式非常一样。

**acquireShared**

```java
public final void acquireShared(int arg) {
  	// 同样调用自定义同步器需要重写的方法，非阻塞式的尝试获取同步状态，如果结果小于零，则获取同步状态失败
    if (tryAcquireShared(arg) < 0)
      	// 调用 AQS 提供的模版方法，进入等待队列
        doAcquireShared(arg);
}
```

**doAcquireShared(arg)**

```java
private void doAcquireShared(int arg) {
  	// 创建共享节点「SHARED」，加到等待队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
      	// 进入“自旋”，这里并不是纯粹意义上的死循环，在独占式已经说明过
        for (;;) {
          	// 同样尝试获取当前节点的前驱节点
            final Node p = node.predecessor();
          	// 如果前驱节点为头节点，尝试再次获取同步状态
            if (p == head) {
              	// 在此以非阻塞式获取同步状态
                int r = tryAcquireShared(arg);
              	// 如果返回结果大于等于零，才能跳出外层循环返回
                if (r >= 0) {
                  	// 这里是和独占式的区别
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

下面是 acquireQueued 和 doAcquireShard 的区别：

可以发现：独占式获取同步状态成功后将自己setHead，而共享式不仅setHead，还会 propagate，也就是传播。

传播什么？可以想到，因为 state 是可以共享的，所以共享锁获取同步状态成功后，还会通知其它线程继续尝试获取同步状态。

![img](https://rgyb.sunluomeng.top/20200613111949.png)

**下面看看这个传播：setHeadAndPropagate**

```java
 // 入参，node： 当前节点
// 入参，propagate：获取同步状态的结果值，即上面方法中的变量 r
private void setHeadAndPropagate(Node node, int propagate) {
   		// 记录旧的头部节点，用于下面的check
       Node h = head; 
   		// 将当前节点设置为头节点
       setHead(node);
       
   		// 通过 propagate 的值和 waitStatus 的值来判断是否可以调用 doReleaseShared 方法
       if (propagate > 0 || h == null || h.waitStatus < 0 ||
           (h = head) == null || h.waitStatus < 0) {
           Node s = node.next;
         	// 如果后继节点为空或者后继节点为共享类型，则进行唤醒后继节点
   				// 这里后继节点为空意思是只剩下当前头节点了，另外这里的 s == null 也是判断空指针的标准写法
           if (s == null || s.isShared())
               doReleaseShared();
       }
   }
```



##### 共享式释放同步状态

**releaseShared**

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

**doReleaseShared**

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
              	// CAS 将头节点的状态设置为0                
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 设置成功后才能跳出循环唤醒头节点的下一个节点
              	unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     // 将头节点状态CAS设置成 PROPAGATE 状态
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

