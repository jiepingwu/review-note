### Semaphore

Semaphore是 一种典型的 **共享式锁** 的AQS实现。

#### Semaphore 源码分析

先看看Semaphore类结构，这里对比一下 独占锁 ReentrantLock。

![img](https://rgyb.sunluomeng.top/20200613135824.png)

发现，简直一摸一样，其实也能理解，毕竟就只是 同步状态是否可以被多个线程同时获取 这一点不同，甚至对于 state 变量来说，就只是判断是否大于0，共享式大于0就可以获取，独占式需要state==0才能获取同步状态。

这里直接提速，查看 **公平锁  和 非公平锁 的尝试获取同步状态** API 的不同。

![img](https://rgyb.sunluomeng.top/20200613140142.png)

一样地，也只是判断是否存在 前驱节点。

继续。

信号量设置的 "初始信号量" 就是我们的 同步状态 state 初始值。

```java
public Semaphore(int permits) {
    	// 默认仍是非公平的同步器，至于为什么默认是非公平的，在上一篇文章中也特意说明过
      sync = new NonfairSync(permits);
  }
  
  NonfairSync(int permits) {
  		super(permits);
  }
```

所以，当我们把 permits 设置为1 时，就是 独占锁。

但 Semaphore 真正的作用肯定不是这样用的，我们通常将 permits 信号量 设置为 >1 的值。

在同一时刻，允许多个(permits个)线程同时访问 一个临界区，这样 能达到 **限流** 的效果。