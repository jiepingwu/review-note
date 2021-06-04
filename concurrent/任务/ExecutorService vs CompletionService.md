![img](https://rgyb.sunluomeng.top/20200810205339.png)

### ExecutorService vs CompletionService

example：假设我们有4个任务来执行复杂的计算，每个任务的执行时间随着输入参数的不同而不同，如果将任务提交到 ExecutorService中。

```java
ExecutorService executorService = Executors.newFixedThreadPool(4);
List<Future> futures = new ArrayList<Future<Integer>>();
futures.add(executorService.submit(A));
futures.add(executorService.submit(B));
futures.add(executorService.submit(C));
futures.add(executorService.submit(D));

// 遍历 Future list，通过 get() 方法获取每个 future 结果
for (Future future:futures) {
    Integer result = future.get();
    // 其他业务逻辑
}
```

而如果使用 CompletionService

```java
ExecutorService executorService = Executors.newFixedThreadPool(4);

// ExecutorCompletionService 是 CompletionService 唯一实现类
CompletionService executorCompletionService= new ExecutorCompletionService<>(executorService );

List<Future> futures = new ArrayList<Future<Integer>>();
futures.add(executorCompletionService.submit(A));
futures.add(executorCompletionService.submit(B));
futures.add(executorCompletionService.submit(C));
futures.add(executorCompletionService.submit(D));

// 遍历 Future list，通过 get() 方法获取每个 future 结果
for (int i=0; i<futures.size(); i++) {
    Integer result = executorCompletionService.take().get();
    // 其他业务逻辑
}
```

对于 ExecutorService，需要等待 A.get() 才能时序后续任务。

而如果是 CompletionService，调用 executorCompletionService.take() 后会返回 最先执行完成的任务结果。

所以，显而易见，如果是 批量执行多个任务，使用completionService。



##### 怎么实现的？：能返回最先执行完的任务结果。

如果用过消息队列，CompletionService的原理很简单：就是将一个 异步任务的生产 和 任务完成结果的消费 解耦。

![img](https://rgyb.sunluomeng.top/20200809210841.png)

简单说，内部存在一个结果队列，哪个任务执行完成了，就将结果加入队列中，这样消费者拿到的就是最先执行完成的结果。



##### CompletionService 源码？(源码还是不看了，看看使用方法)

CompletionService接口，简单的只有5个方法。

```java
Future<V> submit(Callable<V> task);
Future<V> submit(Runnable task, V result);
Future<V> take() throws InterruptedException;
Future<V> poll();
Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
```

submit()能提交 Callable 和 Runnable 任务，并且异步的获取结果。

另外 3 个方法都是从阻塞队列中获取并移除阻塞队列第一个元素，只不过他们的功能略有不同

- Take: 如果**队列为空**，那么调用 **take()** 方法的线程会**被阻塞**
- Poll: 如果**队列为空**，那么调用 **poll()** 方法的线程会**返回 null**
- Poll-timeout: 以**超时的方式**获取并移除阻塞队列中的第一个元素，如果超时时间到，队列还是空，那么该方法会返回 null

**所以说，按大类划分上面5个方法，其实就是两个功能**

- **提交异步任务 （submit）**
- **从队列中拿取并移除第一个元素 (take/poll)**





##### JDK doc 中的两个例子

假设你有一组针对某个问题的solvers，每个都返回一个类型为Result的值，并且想要并发地运行它们，处理每个返回一个非空值的结果，在某些方法使用(Result r)

```java
void solve(Executor e,
           Collection<Callable<Result>> solvers)
    throws InterruptedException, ExecutionException {
    CompletionService<Result> ecs
        = new ExecutorCompletionService<Result>(e);
    for (Callable<Result> s : solvers)
        ecs.submit(s);
    int n = solvers.size();
    for (int i = 0; i < n; ++i) {
        Result r = ecs.take().get();
        if (r != null)
            use(r);
    }
}
```

假设你想使用任务集的第一个非空结果，忽略任何遇到异常的任务，并在第一个任务准备好时取消所有其他任务

```java
void solve(Executor e,
            Collection<Callable<Result>> solvers)
     throws InterruptedException {
     CompletionService<Result> ecs
         = new ExecutorCompletionService<Result>(e);
     int n = solvers.size();
     List<Future<Result>> futures
         = new ArrayList<Future<Result>>(n);
     Result result = null;
     try {
         for (Callable<Result> s : solvers)
             futures.add(ecs.submit(s));
         for (int i = 0; i < n; ++i) {
             try {
                 Result r = ecs.take().get();
                 if (r != null) {
                     result = r;
                     break;
                 }
             } catch (ExecutionException ignore) {}
         }
     }
     finally {
         for (Future<Result> f : futures)
           	// 注意这里的参数给的是 true，详解同样在前序 Future 源码分析文章中
             f.cancel(true);
     }

     if (result != null)
         use(result);
 }
```

