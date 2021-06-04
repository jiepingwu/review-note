### CountDownLatch

#### 前言

并发三大核心：**分工、同步、互斥。**

日常中，会遇到 主线程开启多个子线程并行执行任务，并且主线程需要等到子线程执行完成后汇总，这就会使用到分工和同步。

#### 传统线程同步方式 join

我们使用 join 就能达到 同步的目的。

```java
@Slf4j
public class JoinExample {

	public static void main(String[] args) throws InterruptedException {
		Thread thread1 = new Thread(() -> {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				log.info("Thread-1 执行完毕");
			}
		}, "Thread-1");

		Thread thread2 = new Thread(() -> {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				log.info("Thread-2 执行完毕");
			}
		}, "Thread-2");

		thread1.start();
		thread2.start();

		thread1.join();
		thread2.join();

		log.info("主线程执行完毕");
	}
}
```

![img](https://rgyb.sunluomeng.top/20200626180728.png)

我们来看 join() 的源码，它是怎么阻塞当前调用线程的？

![img](https://rgyb.sunluomeng.top/20200627181448.png)

可以发现，它是不断轮询 join线程是否存活，join线程存活就将 main 线程wait(0)阻塞（即加入Monitor this的等待队列中）。

这里需要解释一下：

其一， join 方法为一个 synchronized 同步方法，monitor 就是 线程实例 this，所以加入this的等待队列中。

其次，可能有疑问：这里为什么 t.join()后，wait(0）会阻塞主线程？跟主线程有什么关系？

其实，会有这种问题是因为对线程的理解不足，我们知道wait()方法会将持有当前锁的线程加入等待队列，那么这个线程是谁？应该是main线程，但是为什么我们会认为是 线程t？

Q：t.join()方法是哪个线程在执行？

A：t.join()方法当然是 调用 t.join() 的线程执行的，也就是 main 线程。所以，这里持有 Monitor this 的就是 main 线程，于是我们 调用wait()时阻塞的当然就是 main 线程。

误区在于：我们把 线程实例调用方法 和 线程执行方法 混淆了，即 t.join() 是由 main 线程执行，而不是 线程t 中执行的。

线程t的执行：我们是调用 t.start() 执行线程的，而 start() 方法调用 native start0() 方法，其中由 OS 创建一个线程 来执行 我们传入的Runnable 中的run() 方法。也就是说，我们要把 线程 t 实例 和 OS中的线程 区别开。

Q：另外，我们在 JDK 中只能看到 wait(0)，那么是在哪里notify()的？

A：在join线程执行完成后，线程的notifyAll()方法会被调用，退出while()循环，继续main线程执行。其中notifyAll在JVM中实现。

##### join 的不足

	1. 我们上面可以看到，轮询检查比较低效，比较消耗CPU资源。
 	2. join还缺少灵活性，比如实际中我们往往不会自己创建线程，而是使用线程池，这就基本不会使用 join 来进行同步。

#### CountDownLatch

​	那么为了线程间同步（或者说为了是 一个线程等待其它线程执行完成），怎么实现呢？我们可以使用什么来代替 join？

​	我们就可以使用 CountDownLatch，直译过来就是 "门闩"。

​	意义上比较好理解：一个门闩，一个线程执行完调用countDown()，门闩上的门闩数就减一，直到门闩数为0时，调用countDownLatch.await()等待门开的阻塞线程才会继续执行。

**example**

```java
@Slf4j
public class CountDownLatchExample {

	private static CountDownLatch countDownLatch = new CountDownLatch(2);

	public static void main(String[] args) throws InterruptedException {
		// 这里不推荐这样创建线程池，最好通过 ThreadPoolExecutor 手动创建线程池
		ExecutorService executorService = Executors.newFixedThreadPool(2);

		executorService.submit(() -> {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				log.info("Thread-1 执行完毕");
				//计数器减1
				countDownLatch.countDown();
			}
		});

		executorService.submit(() -> {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				log.info("Thread-2 执行完毕");
				//计数器减1
				countDownLatch.countDown();
			}
		});

		log.info("主线程等待子线程执行完毕");
		log.info("计数器值为：" + countDownLatch.getCount());
		countDownLatch.await();
		log.info("计数器值为：" + countDownLatch.getCount());
		log.info("主线程执行完毕");
		executorService.shutdown();
	}
}
```

**官方示例1**

主线程和子线程之间的协同，子线程开启后等待主线程预处理后通知(拉下门闩startSignal)，主线程等待所有子线程任务执行完成。

```java
class Driver {
    void main() throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(N);	// N 为子线程数量
        
        for (int i = 0; i < N; ++i) { // 创建并且开启线程
            new Thread(new Worker(startSignal, doneSignal)).start();
        }
        
        doSomethingElse(); // 其它预处理任务，任务线程启动前的任务执行。
        startSignal.countDown(); // startSignal门闩countDown 使所有任务线程开始执行
        doSomethingElse();
        doneSignal.await(); // 等待所有任务线程执行完成
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;
    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }
    
    @override
	public void run() {
        try {
            startSignal.await(); // 等待预加载任务执行完成后 拉下打开startSignal门闩
            doWork();
            doneSignal.countDown(); // 任务完成，doneSignal门闩减一
        } catch (InterruptedException ex) {} // return
    }
    
    void doWork() { ... }
}
```

**官方示例2**

分治的场景，将一个问题分为 N 个部分，(比如将一个大的list拆分为多个部分，一个worker处理一个部分)，worker执行完成后countDown，当所有线程执行完成后，Driver主线程才往下继续执行。

```java
class Driver {
    void main() throws InterruptedException {
        CountDownLatch doneSignal = new CountDownLatch(N);
        Executor e = ...;
        
        // 创建并且执行线程
        for (int i = 0; i < N; ++i) {
            e.execute(new WorkerRunnable(doneSignal, i));
        }
        doneSignal.await(); // 等待所有线程完成
    }
}

class WorkerRunnable implements Runnable {
	private final CountDownLatch doneSignal;
    private final int i;		// task index
    WorkerRunnable(CountDownLatch doneSignal, int i) {
        this.doneSignal = doneSignal;
        this.i = i;
    }
    @override
    public void run() {
        try {
            doWork(i);
            doneSignal.countDown(); // count down
        } catch(InterruptedException ex) { } // return
    }
    void doWork() { ... }
}
```



#### CountDownLatch 源码实现

![img](https://rgyb.sunluomeng.top/20200626195016.png)

可见，CountDownLatch 也是基于 共享锁 实现，传入的 Latch 门闩数量 就是 同步状态的值。

![img](https://rgyb.sunluomeng.top/20200626195503.png)

CountDownLatch 对外部暴露的 是 await 和 countDown 方法，其中 await 调用 tryAcquireShared，countDown 调用 tryReleaseShared 。

**await**

await()方法可以抛出 InterruptedException ，所以它是可以响应中断的。

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

其内部调用 模板方法 - acquireSharedInterruptibly()

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
  	// 如果监测到中断标识为true,会重置标识，然后抛出 InterruptedException
    if (Thread.interrupted())
        throw new InterruptedException();
  	// 调用重写的 tryAcquireShared 方法，该方法结果如果大于零则直接返回，程序继续向下执行，如果小于零，则会阻塞自己
    if (tryAcquireShared(arg) < 0)
      	// state不等于0，则尝试阻塞自己
        doAcquireSharedInterruptibly(arg);
}
```

其中，重写的 tryAcquireShared 方法很简单，判断同步状态 state 是否为0，为0返回1(表示所有latch门闩放下，子线程执行完成)，否则返回-1。

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

如果state!=0，即 latch 门闩还没有全部打开，子线程没有全部执行完成，则 调用 doAcquireSharedInterruptibly(arg) 方法阻塞自己。

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
              	// 再次尝试获取同步状态，如果大于0，说明子线程全部执行完毕，直接返回
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);	// 将自己设为头节点，propagate广播通知所有后继线程节点抢锁
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
          	// 阻塞自己
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**countDown**

```java
public void countDown() {
    sync.releaseShared(1);
}
```

同样调用模板方法 releaseShared

```java
public final boolean releaseShared(int arg) {
  	// 调用自己重写的同步器方法
    if (tryReleaseShared(arg)) {
      	// 唤醒调用 await() 被阻塞的线程
        doReleaseShared();
        return true;
    }
    return false;
}
```

调用重写的方法 tryReleaseShared

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
      	// 如果当前状态值为0，则直接返回 （1）
        if (c == 0)
            return false;
      	// 使用 CAS 让计数器的值减1 （2）
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

​	如果state == 0，直接返回false，门闩已经没有闩了，不需要释放。否则CAS+死循环 修改为 state--，返回 是否 state == 0。

**await(long timeout, TimeUnit unit)**

```java
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

 	同样可以响应中断，其实是await()的可超时版本。

​	主线程设置超时时间，如果改时间内子线程没有执行完成，主线程就直接返回。

#### 小结

CountDownLatch 有一个问题：就是 门闩不可复用，当 state值减到0后，不能再将其复原为初始值，如果此时再有线程调用 await() 方法，该线程不会阻塞，直接返回，不能再起到等待其它线程执行结果 的同步作用了。

怎么办？有没有可以复用的？有，那就是 CyclicBarrier。