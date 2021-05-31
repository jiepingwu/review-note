### Condition

首先 Condition 对标的是 Synchronized 内置锁 中的 wait 和 notify 方法。也就是 jvm 中的 MESA 监视器模型。

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host/blog-img20200315110223.png)

Condition也是一个接口，也是需要有实现类的。

![img](https://rgyb.sunluomeng.top/20200530200503.png)

故事就从 lock.newCondition 方法说起。

```java
public Condition newCondition() {
	// 使用自定义的条件
	return sync.newCondition();
}
```

自定义同步器封装了该方法

```java
Condition newCondition() {
	return new ConditionObject();
}
```

ConditionObject 就是 Condition 的实现类，该类就定义在 AQS 中，只有两个成员变量。

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

我们只需要查看 await and signal 方法是怎么使用这两个成员变量的就行了。

**await**

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
  	// 同样构建 Node 节点，并加入到等待队列中
    Node node = addConditionWaiter();
  	// 释放同步状态
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {	// 轮询node节点是否在 同步队列中，不在，就park当前线程。
      	// 挂起当前线程
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

这里同样的构造node节点，不过与 AQS 中的 addWaiter 加入 **同步队列** 区别，这里的 addConditionWaiter()方法将 node节点 加入 Condition 维护的 **等待队列中。**

**这里会问：为什么 将节点加入 等待队列的过程中，不需要使用 CAS 插入节点？**

因为这里，像synchronized内置锁一样，是获取到了锁才会调用 await 和 signal 方法，所以，是获取到了锁，就不需要使用 CAS 保证并发插入节点的安全了。

AQS 中 同步队列 和 等待队列 的结构 如下图。其中，同步队列是一个双向链表，而 等待队列是多个单链表。并且存在head and tail 头尾指针方便插入和移除。

![img](https://rgyb.sunluomeng.top/20200530205315.png)

**signal and signalAll**

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

signal 方法调用 doSingal(firstWaiter) ，唤醒条件等待队列中的 头节点。

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
      	// 调用该方法，将条件等待队列的线程节点移动到同步队列中
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

transferForSignal(first)

```java
final boolean transferForSignal(Node node) {       
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

   	// 重新进行入队操作
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
      	// 唤醒同步队列中该线程
        LockSupport.unpark(node.thread);
    return true;
}
```

![image-20210531150901465](..\references-figures\image-20210531150901465.png)

​	signal时将等待队列中的头节点移动到 同步队列的尾部，并且unpark 该节点线程。

**signalAll**

​	signalAll就很简单了，从 等待队列头节点开始，循环判断是否还有nextWaiter，有的话就像 signal 一样移动到 同步队列尾部并且unpark。

```java
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

