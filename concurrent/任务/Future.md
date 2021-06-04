![img](https://rgyb.sunluomeng.top/20200705203141.png)

### Futrue

#### Runnable 创建线程的短板

- 继承Thread类
- 实现Runnable接口

这两种方式都 是 三无方法：1. 不能传入参数  2. 不能抛出异常 3. 不能得到返回值

于是，Callable 出现了。

#### Callable

```java
 /**
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> the result type of method {@code call}
 */
@FunctionalInterface
public interface Callable<V> {
    
    V call() throws Exception;
}
```

callable 是一个泛型函数式接口，其中只有一个 call() 方法。该方法可以返回 泛型值 V。

#### Runnable vs Callable

- 执行机制

Runnable 既可以用在 Thread 类中，也可以在 ExecutorService类中配合线程池使用。

但是，Callable 只能在 ExecutorService 中使用，你在 Thread 类中完全找不到 Callable。

- 异常处理

Runnable不能抛出异常，而 Callable可以。

![img](https://rgyb.sunluomeng.top/20200705122813.png)

![img](https://rgyb.sunluomeng.top/20200704185918.png)

ExecutorService 的 execute(Runnable) 只能执行 runnable 任务，而 submit() 则可以提交 Callable 和 Runnable 任务，并且都有 Future 类型的返回值。

```java
void execute(Runnable command);

<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

#### Future

![img](https://rgyb.sunluomeng.top/20200704193123.png)

```java
// 取消任务
boolean cancel(boolean mayInterruptIfRunning);

// 获取任务执行结果
V get() throws InterruptedException, ExecutionException;

// 获取任务执行结果，带有超时时间限制
V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException,  TimeoutException;

// 判断任务是否已经取消
boolean isCancelled();

// 判断任务是否已经结束
boolean isDone();
```

调用 get() 方法时，如果计算结果被取消了，则抛出 CancellationException （具体原因，你会在下面的源码分析中看到）

#### FutureTask

![img](https://rgyb.sunluomeng.top/20200705100500.png)

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

FutureTask 接口 实现了 RunnableFuture 接口， 而 RunnableFuture 接口同时实现了 Runnable 和 Future 接口。

所以，FutureTask 同时具有 Runnable 可以用在 ExecutorService 中配合线程池使用；Future 可以得到执行结果的特性。



##### FutureTask 源码

Callable才可以得到返回值，Runnable不行，FutureTask内部封装了一个Callable。

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

```java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

可以看到，即使传入的是 Runnable，也会 通过 Executors.callable(runnable, result) 转换为 Callable 类型。



```java
public void run() {
  	// 如果状态不是 NEW，说明任务已经执行过或者已经被取消，直接返回
  	// 如果状态是 NEW，则尝试把执行线程保存在 runnerOffset（runner字段），如果赋值失败，则直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
      	// 获取构造函数传入的 Callable 值
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
              	// 正常调用 Callable 的 call 方法就可以获取到返回值
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
              	// 保存 call 方法抛出的异常
                setException(ex);
            }
            if (ran)
              	// 保存 call 方法的执行结果
                set(result);
        }
    } finally {        
        runner = null;       
        int s = state;
      	// 如果任务被中断，则执行中断处理
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

- **几个问题：**
- FutureTask 是怎样在 run() 方法中获取返回值的？
- 它将返回值放到哪里了？
- get() 方法又是怎样拿到这个返回值的呢？

将 call() 的返回值 result，调用set(result) 保存在 一个 Object outcome 中。异常也是一样，会调用setException(ex)保存异常对象。

**由于outcome是成员变量，需要保证线程安全，使用CAS的方式修改state变量的值，并且利用state的volatile读写 带来 outcome 变量的可见性。**

```java
/** The result to return or exception to throw from get() */
private Object outcome; // non-volatile, protected by state reads/writes

// 保存异常结果
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        finishCompletion();
    }
}

// 保存正常结果
protected void set(V v) {
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    outcome = v;
    UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
    finishCompletion();
  }
}
```

**get()方法**

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
  	// 如果 state 还没到 set outcome 结果的时候，则调用 awaitDone() 方法阻塞自己
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
  	// 返回结果
    return report(s);
}
```

**axaitDone**

```java
// get 方法支持超时限制，如果没有传入超时时间，则接受的参数是 false 和 0L
// 有等待就会有队列排队或者可响应中断，从方法签名上看有 InterruptedException，说明该方法这是可以被中断的
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
  	// 计算等待截止时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
      	// 如果当前线程被中断，如果是，则在等待对立中删除该节点，并抛出 InterruptedException
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
      	// 状态大于 COMPLETING 说明已经达到某个最终状态（正常结束/异常结束/取消）
      	// 把 thread 只为空，并返回结果
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
      	// 如果是COMPLETING 状态（中间状态），表示任务已结束，但 outcome 赋值还没结束，这时主动让出执行权，让其他线程优先执行（只是发出这个信号，至于是否别的线程执行一定会执行可是不一定的）
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
      	// 等待节点为空
        else if (q == null)
          	// 将当前线程构造节点
            q = new WaitNode();
      	// 如果还没有入队列，则把当前节点加入waiters首节点并替换原来waiters
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
      	// 如果设置超时时间
        else if (timed) {
            nanos = deadline - System.nanoTime();
          	// 时间到，则不再等待结果
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
          	// 阻塞等待特定时间
            LockSupport.parkNanos(this, nanos);
        }
        else
          	// 挂起当前线程，知道被其他线程唤醒
            LockSupport.park(this);
    }
}
```

总的来说，进入这个方法，通常会经历三轮循环

1. 第一轮for循环，执行的逻辑是 `q == null`, 这时候会新建一个节点 q, 第一轮循环结束。
2. 第二轮for循环，执行的逻辑是 `!queue`，这个时候会把第一轮循环中生成的节点的 next 指针指向waiters，然后CAS的把节点q 替换waiters, 也就是把新生成的节点添加到waiters 中的首节点。如果替换成功，queued=true。第二轮循环结束。
3. 第三轮for循环，进行阻塞等待。要么阻塞特定时间，要么一直阻塞知道被其他线程唤醒。

对于第二轮循环，大家可能稍稍有点迷糊，我们前面说过，有阻塞，就会排队，有排队自然就有队列，FutureTask 内部同样维护了一个队列

```
private volatile WaitNode waiters;
```

说是等待队列，其实就是一个 Treiber 类型 stack，既然是 stack， 那就像手枪的弹夹一样（脑补一下子弹放入弹夹的情形），后进先出，所以刚刚说的第二轮循环，会把新生成的节点添加到 waiters stack 的首节点。



##### 谁来唤醒？

如果程序运行正常，通常调用 get() 方法，会将当前线程挂起，那谁来唤醒呢？自然是 run() 方法运行完会唤醒，设置返回结果（set方法）/异常的方法(setException方法) 两个方法中都会调用 **finishCompletion** 方法，该方法就会唤醒等待队列中的线程

```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                  	// 唤醒等待队列中的线程
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```



**cancel**

查看 Future cancel()，该方法注释上明确说明三种 cancel 操作一定失败的情形

1. 任务已经执行完成了
2. 任务已经被取消过了
3. 任务因为某种原因不能被取消

其它情况下，cancel操作将返回true。值得注意的是，cancel操作返回 true 并不代表任务真的就是被取消, **这取决于发动cancel状态时，任务所处的状态**

- 如果发起cancel时任务还没有开始运行，则随后任务就不会被执行；
- 如果发起cancel时任务已经在运行了，则这时就需要看**mayInterruptIfRunning**参数了：
    - 如果mayInterruptIfRunning 为true, 则当前在执行的任务会被中断
    - 如果mayInterruptIfRunning 为false, 则可以允许正在执行的任务继续运行，直到它执行完

```java
public boolean cancel(boolean mayInterruptIfRunning) {
  
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
      	// 需要中断任务执行线程
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
              	// 中断线程
                if (t != null)
                    t.interrupt();
            } finally { // final state
              	// 修改为最终状态 INTERRUPTED
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
      	// 唤醒等待中的线程
        finishCompletion();
    }
    return true;
}
```

