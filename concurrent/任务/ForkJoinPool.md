### ForkJoinPool

#### 前言

总结一下关于任务的工具类：

- ExecutorService 和 Future ：我们可以使用线程池管理线程将一个大任务拆分成多个子任务来执行，借助 Futrue 能获取子任务执行的执行结果-**两者结合可以处理简单的并行任务。**
- CompletableFuture：借助CompletableFuture 降低了异步编程的难度，我们可以使用CompletableFuture 实现异步执行任务，任务完成后执行回调方法，并且使用CompletableFuture可以使用流式编程使用AND OR 聚合 等方式编排任务执行顺序。
- CompletionService：因为任务执行完成有先后顺序，使用 CompletionService 可以在批量执行任务时，拿到 最先完成的任务执行结果。(内部将 任务的提交 和 任务执行完成结果的消费 解耦，任务完成后加入消费队列等待消费者消费。)

下图展示上面3种场景：

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20201228204001.png)

但是，还没有结束，这就是并发编程的魅力，对于上面的使用来说，子任务不可能再分为多个任务执行，也就是说，子任务不管有多繁重，子线程都只能完全执行完。

如果，我们能把一个任务拆分为多个更小的子问题，直到子问题简单到可以直接求解就好了。这就是分治。

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20201228211510.png)

#### ForkJoin

有子任务，就需要用到多线程。而 **执行子任务的线程不允许单独创建，要用线程池管理。**所以，ForkJoin 框架中就出现了 ForkJoinPool 和 ForkJoinTask。简单类比：ForkJoinPool ~= ThreadPoolExecutor

##### ForkJoinTask

```java
 /**
 * Abstract base class for tasks that run within a {@link ForkJoinPool}.
 * A {@code ForkJoinTask} is a thread-like entity that is much
 * lighter weight than a normal thread.  Huge numbers of tasks and
 * subtasks may be hosted by a small number of actual threads in a
 * ForkJoinPool, at the price of some usage limitations.
 *
 * @since 1.7
 * @author Doug Lea
 */
public abstract class ForkJoinTask<V> implements Future<V>, Serializable
```

ForkJoinTask 实现了 Future 接口。类如其名，ForkJoinTask 中两个核心方法 **fork() and join()**

- fork()：异步执行一个子任务(拆分)
- join()：阻塞当前线程等待子任务执行结果(合并)

另外，ForkJoinTask 是一个 abstract class，它有两个抽象子类 **RecursiveAction 和 RecursiveTask**

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20201228221139.png)

**RecursiveAction 和 RecursiceTask 的区别**

发现：Action没有返回值(这跟命名也相似吧，action就应该没有返回值)，Task则是有result返回值。

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20201228221929.png)

```java
public abstract class RecursiveAction extends ForkJoinTask<Void>{
	...
  /**
   * The main computation performed by this task.
   */
  protected abstract void compute();
  ...
}



public abstract class RecursiveTask<V> extends ForkJoinTask<V>{
	...
  protected abstract void compute();
  ...
}
```

两个抽象类中都定义了 **compute()** 方法，需要子类提供具体实现。

关键是：子类应该怎么实现这个方法 compute()。

遵循分治思想，回答三个问题：

- 什么时候进一步拆分任务？
- 什么时候满足最小可执行任务，即不再拆分？
- 什么时候汇总子任务结果？

伪代码就是：

```java
if (任务小到不需要继续拆分) {
    直接计算得到结果
} else {
    拆分子任务
    调用子任务的fork()进行计算
    调用子任务的join()合并计算结果
}
```

##### ForkJoin 来计算 Fibonacci(n)

```java
public class ForkJoinDemo {
    public static void main(String[] args) {
        int n = 20;
        
        ForkJoinPool forkJoinPool = new ForkJoinPool(4, ForkJoinPool.defaultForkJoinWorkerThreadFactory, null, false);
        
        Fibonacci fibonacci = new Fibonacci(n);
        
        Integer result = forkJoinPool.invoke(fibonacci);
        System.out.println("Fibonacci {} 的结果是 {}", n, result);
    }
}

class Fibonacci extends RecursiveTask<Integer> {
    final int n;
    Fibonacci(int n) {
        this.n = n;
    }
    
    @Override
    public Integer compute() {
		if (n <= 1) return n; // 类似递归，可计算的最小单元，直接计算返回。
        Fibonacci f1 = new Fibonacci(n - 1);  // 拆分为子任务
        f1.fork();
        Fibonacci f2 = new Fibonacci(n - 2);
        return f2.compute() + f1.join(); // f1.join 等待子任务的执行结果
    }
}
```

##### ForkJoinPool

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20201230205543.png)

先考虑，为什么有了 ThreadPoolExecutor 还需要有 ForkJoinPool ? 或者说 ThreadPoolExecutor 和 ForkJoinPool 有什么区别？

##### ForkJoinPool 和 ThreadPoolExecutor 的区别

###### **ThreadPoolExecutor**

先看ThreadPoolExecutor，我们已经知道，ThreadPoolExecutor模型实现其实是基于 生产者消费者模型的，即 线程集合中的线程不断消费 任务队列中的任务，任务的提交和线程的执行解耦，调用者只需要将任务提交给线程池即可。

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20201230215609.png)

而对于 分治思想来说，任务之间会存在父子依赖关系，所以用 ThreadPoolExecutor 来处理就不太现实，于是 ForkJoinPool 用于功能补充实现了。

###### **ForkJoinPool**

也就是说，ForkJoinPool 考虑了 任务间的父子依赖关系，让线程执行属于自己的任务task。

所以，**相比于 ThreadPoolExecutor 内部维护的单个Task Queue，ForkJoinPool 内部维护了多个 任务队列 Task Queue。**

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20210102222146.png)

存在多个 Task Queue，ForkJoinPool 维护的 WorkQueue[] 就是一个 Queue 数组。

那么，**任务怎么提交？怎么确定任务提交到哪一个队列中？**



###### ForkJoinPool 中的 Router Rule

路由规则，任务的提交有两种

1. 从外部直接提交的(submission task)
2. 任务自己fork出来的(worker task)

为了区分这两种不同的task，Doug Lea设计了一个简单的路由规则(Router Rule)：

- 将 submission task 放到 WorkQueue 数组的 【偶数】下标中。
- 将 worker task 放到 WorkQueue 数组的 【奇数】下标中，并且只有奇数下标才有线程 Worker 与之相对应。

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20210217101059.png)



###### work-stealing 工作窃取

每个任务的执行时间是不一样的，执行快的线程执行的任务队列可能就是空的，为了最大化的利用CPU的资源，就允许空闲线程去其它队列中获取任务，这就是 ForkJoinPool 中的 work-stealing 机制。

**并发问题**

当前线程执行一个任务，可能还有其它线程来stealing work，就会产生竞争，为了减少竞争，**WorkQueue 被设计成一个 双端队列**。

- 支持 LIFO 的 push 和 pop 操作- 操作 top 端。
- 支持 FIFO 的 poll 操作- 操作 base 端。

Worker 线程 在操作自己的 WorkQueue 的时候默认是 LIFO (可选 FIFO)，当 其它线程 Worker 尝试 stealing-work 的时候，执行的是 FIFO 操作，即 从 base 端 stealing。简单来说：**执行自己的任务时，从队列尾部开始执行(top端)，而其它Worker-stealing的时候，从队列头部(base端)stealing，看下图**。

![img](https://cdn.jsdelivr.net/gh/FraserYu/img-host@master/blog-img20210103103303.png)

这样做就带来两个好处：

1. LIFO操作只有对应的Worker才能执行，push 和 pop 不需要考虑并发。
2. 拆分时，越大的任务越在 WorkQueue 的 base 端，可以尽早地分解，能尽快进入计算。

**疑问还很多：**

- 有竞争就有锁，ForkJoinPool 如何控制状态的？
- ForkJoinPool 的线程数怎么控制的？
- 上面说的 Router Rule 的具体逻辑是什么？



##### ForkJoin 源码分析

###### todo 

ref：https://dayarch.top/p/java-fork-join-pool.html