### CyclicBarrier

##### example

```java
@Slf4j
public class CyclicBarrierExample {

   // 创建 CyclicBarrier 实例，计数器的值设置为2
   private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

   public static void main(String[] args) {
      ExecutorService executorService = Executors.newFixedThreadPool(2);
      int breakCount = 0;

     	// 将线程提交到线程池
      executorService.submit(() -> {
         try {
            log.info(Thread.currentThread() + "第一回合");
            Thread.sleep(1000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第二回合");
            Thread.sleep(2000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第三回合");
         } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
         } 
      });

      executorService.submit(() -> {
         try {
            log.info(Thread.currentThread() + "第一回合");
            Thread.sleep(2000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第二回合");
            Thread.sleep(1000);
            cyclicBarrier.await();

            log.info(Thread.currentThread() + "第三回合");
         } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
         }
      });

      executorService.shutdown();
   }

}
```

![img](https://rgyb.sunluomeng.top/20200627152822.png)

1. 怎么判断所有线程都到达屏障点的？
2. 突破某一屏障后，又是怎么重置 CyclicBarrier 计数器，等待线程再一次突破屏障呢？

![](https://rgyb.sunluomeng.top/20200627163939.png)

1. await() 方法，猜测应该和 CountDownLatch 是类似的，都是获取同步状态，阻塞自己
2. ReentrantLock，CyclicBarrier 内部竟然也用到了我们之前讲过的 ReentrantLock，猜测这个锁一定保护 CyclicBarrier 的某个变量，那肯定也是基于 AQS 相关知识了
3. Condition，存在条件，猜测会有等待/通知机制的运用



##### 下面看看源码

```java
private final int parties;
private int count;

public CyclicBarrier(int parties) {
    this(parties, null);
}

    /**
     * Creates a new {@code CyclicBarrier} that will trip when the
     * given number of parties (threads) are waiting upon it, and which
     * will execute the given barrier action when the barrier is tripped,
     * performed by the last thread entering the barrier.
     *
     * @param parties the number of threads that must invoke {@link #await}
     *        before the barrier is tripped
     * @param barrierAction the command to execute when the barrier is
     *        tripped, or {@code null} if there is no action
     * @throws IllegalArgumentException if {@code parties} is less than 1
     */
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

根据注释说明，parties 代表冲破屏障之前要触发的线程总数，count 本身又是计数器，那问题来了

> 直接就用 count 不就可以了嘛？为啥同样用于初始化计数器，要维护两个变量呢？

从 parties 和 count 的变量声明中，你也能看出一些门道，前者有 final 修饰，初始化后就不可以改变了，因为 CyclicBarrier 的设计目的是可以循环利用的，所以始终用 parties 来记录线程总数，当 count 计数器变为 0 后，如果没有 parties 的值赋给它，怎么进行重新复用再次计数呢，所以这里维护两个变量很有必要

**await**

```java
// 从方法签名上可以看出，该方法同样可以被中断，另外还有一个 BrokenBarrierException 异常，我们一会看
public int await() throws InterruptedException, BrokenBarrierException {
    try {
      	// 调用内部 dowait 方法， 第一个参数为 false，表示不设置超时时间，第二个参数也就没了意义
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

**-dowait(boolean timed, long nanos)**

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    // 还记得之前说过的 Lock 标准范式吗？ JDK 内部都是这么使用的，你一定也要遵循范式
    lock.lock();
    try {
        final Generation g = generation;

      	// broken 是静态内部类 Generation唯一的一个成员变量，用于记录当前屏障是否被打破，如果打破，则抛出 BrokenBarrierException 异常
      	// 这里感觉挺困惑的，我们要【冲破】屏障，这里【打破】屏障却抛出异常，注意我这里的用词
        if (g.broken)
            throw new BrokenBarrierException();

      	// 如果线程被中断，则会通过 breakBarrier 方法将 broken 设置为true，也就是说，如果有线程收到中断通知，直接就打破屏障，停止 CyclicBarrier， 并唤醒所有线程
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
      
      	// ************************************
      	// 因为 breakBarrier 方法在这里会被调用多次，为了便于大家理解，我直接将 breakBarrier 代码插入到这里
      	private void breakBarrier() {
          // 将打破屏障标识 设置为 true
          generation.broken = true;
          // 重置计数器
          count = parties;
          // 唤醒所有等待的线程
          trip.signalAll();
    		}
      	// ************************************

				// 每当一个线程调用 await 方法，计数器 count 就会减1
        int index = --count;
      	// 当 count 值减到 0 时，说明这是最后一个调用 await() 的子线程，则会突破屏障
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
              	// 获取构造函数中的 barrierCommand，如果有值，则运行该方法
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
              	// 激活其他因调用 await 方法而被阻塞的线程，并重置 CyclicBarrier
                nextGeneration();
              
                // ************************************
                // 为了便于大家理解，我直接将 nextGeneration 实现插入到这里
                private void nextGeneration() {
                    // signal completion of last generation
                    trip.signalAll();
                    // set up next generation
                    count = parties;
                    generation = new Generation();
                }
                // ************************************
              
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

      	// index 不等于0， 说明当前不是最后一个线程调用 await 方法
        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
              	// 没有设置超时时间
                if (!timed)
                  	// 进入条件等待
                    trip.await();
                else if (nanos > 0L)
                  	// 否则，判断超时时间，这个我们在 AQS 中有说明过，包括为什么最后超时阈值 spinForTimeoutThreshold 不再比较的原因，大家会看就好
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
              	// 条件等待被中断，则判断是否有其他线程已经使屏障破坏。若没有则进行屏障破坏处理，并抛出异常；否则再次中断当前线程

                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

          	// 如果新一轮回环结束，会通过 nextGeneration 方法新建 generation 对象
            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

doWait 就是 CyclicBarrier 的核心逻辑， 可以看出，该方法入口使用了 ReentrantLock，这也就是为什么 Generation broken 变量没有被声明为 volatile 类型保持可见性，因为对其的更改都是在锁的内部，同样在锁的内部对计数器 count 做更新，也保证了原子性

doWait 方法中，是通过 nextGeneration 方法来重新初始化/重置 CyclicBarrier 状态的，该类中还有一个 reset() 方法，也是重置 CyclicBarrier 状态的

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

但 reset() 方法并没有在 CyclicBarrier 内部被调用，显然是给 CyclicBarrier 使用者来调用的，那问题来了

> **什么时候调用 reset() 方法呢**

正常情况下，CyclicBarrier 是会被自动重置状态的，从 reset 的方法实现中可以看出调用了 breakBarrier方法，也就是说，**调用 reset 会使当前处在等待中的线程最终抛出 BrokenBarrierException 并立即被唤醒，所以说 reset() 只会在你想打破屏障时才会使用**



##### 构造器中传入的 Runnable barrierAction

上述示例，我们构建 CyclicBarrier 对象时，并没有传递 barrierAction对象， 我们修改示例传入一个 barrierAction对象，看看会有什么结果：

```
复制// 创建 CyclicBarrier 实例，计数器的值设置为2
private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2, () -> {
   log.info("全部运行结束");
});
```

运行结果：

[![img](https://rgyb.sunluomeng.top/20200627180315.png)](https://rgyb.sunluomeng.top/20200627180315.png)

从运行结果中来看，每次冲破屏障后都会执行 CyclicBarrier 初始化 barrierAction的方法， 这与我们对 doWait() 方法的分析完全吻合，从上面的运行结果中可以看出，最后一个线程是运行 barrierAction run() 方法的线程，我们再来形象化的展示一下整个过程

![img](https://rgyb.sunluomeng.top/20200627203542.png)

从上图可以看出，barrierAction 与每次突破屏障是串行化的执行过程，假如 barrierAction 是很耗时的汇总操作，那这就是可以优化的点了，我们继续修改代码，将 barrierAction任务交给另一个线程完成，而不是最后这个线程。

```java
// 创建单线程线程池
private static Executor executor = Executors.newSingleThreadExecutor();

// 创建 CyclicBarrier 实例，计数器的值设置为2
private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2, () -> {
   executor.execute(() -> gather());
});

private static void gather() {
   try {
      Thread.sleep(2000);
   } catch (InterruptedException e) {
      e.printStackTrace();
   }
   log.info("全部运行结束");
}
```

我们这里将 CyclicBarrier 的回调函数 barrierAction使用单线程的线程池，这样最后一个冲破屏障的线程就不用等待 barrierAction 的执行，直接分配个线程池里的线程异步执行，进一步提升效率

![img](https://rgyb.sunluomeng.top/20200627205306.png)