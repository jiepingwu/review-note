### ReentrantLock

ReentrantLock 是独占锁的实现，我们就来看看 ReentrantLock 是如何使用 AQS 的。

![img](https://rgyb.sunluomeng.top/20200530113417.png)

其中 Sync 继承自 AQS，而 FairSync 和 NonfairSync 提供了 公平 和 非公平 的实现。

![img](https://rgyb.sunluomeng.top/20200531100921.png)



##### 公平锁 和 非公平锁

公平锁就是判断同步队列中是否还有前驱节点的存在，只有没有前驱节点才能获取锁；

非公平锁则是不管是否存在前驱节点，都可以去获取同步状态；

![img](https://rgyb.sunluomeng.top/20200531102752.png)

​	图中看出，区别在于 tryAcquire(int acquires) 方法中，获取同步状态c时，如果 c == 0 表示可以获取同步状态，此时 公平锁 会检查 hasQueuedPredecessors() 是否存在前驱节点，如果存在，则不会去获取同步状态。而 nonfairTryAcquire(int acquires)中没有这个检查。



**为什么要有公平锁 和 非公平锁 两种实现？**

原因1：恢复挂起的线程 到 真正地获取到锁 存在时间差，所以非公平锁能更充分地利用 CPU 时间片，减少 CPU 空闲状态时间。

原因2：如果采用非公平锁，释放同步状态时不需要考虑是否存在前驱节点，所以刚刚释放锁的线程再次获取同步状态的概率非常大，减少了 线程切换的开销。



于是，除了不需要公平锁带来的公平的特性，非公平锁也更高效，ReentrantLock 中的默认构造器为 非公平锁AQS。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```

那么，非公平锁会存在什么问题？

显而易见的，非公平锁，忽略了排序，导致可能排队的长时间排队，也没有机会获取到锁，这就是 "饥饿"。



**如何选择 公平锁 和 非公平锁？**

如果为了更高的吞吐量，应该选择 非公平锁，因为如果减少了线程切换的时间，吞吐量自然就上去了。

否则，就可以用 公平锁 还大家一个公平。



**可重入锁**

为什么要支持可重入锁？

如果线程已经获取到同步状态，第二次获取时会被自己阻塞。那不是很奇怪吗？这样递归是没法实现的。

所以synchronized是支持锁重入的，怎么支持的？使用 state 计数，重入时 state++，只有state=0时才能release（moniterexit)。

同样的，ReentrantLock 也支持锁重入，一样使 同步状态 state++，释放锁时state--直到等于0。

