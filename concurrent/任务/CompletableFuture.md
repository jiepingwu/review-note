### ![img](https://rgyb.sunluomeng.top/20200719211506.png)

### CompletableFuture

#### Future的短板

Future 虽然解决了 Runnable 的 "三无" 问题，但是还是存在短板，比如：

1. 不能手动完成计算：使用Future 允许子线程来计算时，不能手动结束计算然后返回前一个版本的计算结果。

2. get()获取结果时阻塞：Future 不会通知你它的完成，调用get方法时会阻塞到任务执行完成，没有办法使用回调方法。

3. 不能链式执行：不能完成多任务的链式执行，更多任务时代码修改量大。

4. 整和多个 Future 执行结果方式笨重：如果 多个Future并行执行，需要在这些任务全部完成后执行后续操作，Future本身是无法实现的，需要借助工具类 Executors 的方法。

    ```java
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
    ```

5. 没有异常处理。

##### CompletableFuture

![img](https://rgyb.sunluomeng.top/20200719153822.png)

实现了 Future 接口，也就是实现 get(), get(long timeout, TimeUnit unit), cancel(), isCancelled(), isDone() 方法。

实现了 CompletionStage 接口。

这个接口直译为 "竣工阶段"，描述的是线程执行的方式，任务的时序关系。

![img](https://rgyb.sunluomeng.top/20200719181233.png)

CompletableFuture 大约有 50 种不同处理串行、并行、组合以及处理错误的方法。

##### 串行 then

```java
CompletableFuture<Void> thenRun(Runnable action)
CompletableFuture<Void> thenRunAsync(Runnable action)
CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)
  
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
  
CompletableFuture<Void> thenAccept(Consumer<? super T> action) 
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)
  
<U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)  
<U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn)
<U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor)
```

##### 聚合 AND 

##### `combine... with...` 和 `both...and...` 都是要求两者都满足，也就是 and 的关系了

```java
<U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
<U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)

<U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
<U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
<U> CompletableFuture<Void> thenAcceptBothAsync( CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action, Executor executor)
  
CompletableFuture<Void> runAfterBoth(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

##### 聚合 OR

```java
<U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn)
<U> CompletableFuture<U> applyToEitherAsync(、CompletionStage<? extends T> other, Function<? super T, U> fn)
<U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn, Executor executor)

CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)

CompletableFuture<Void> runAfterEither(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action)
CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

##### 异常处理

```java
CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn)
CompletableFuture<T> exceptionallyAsync(Function<Throwable, ? extends T> fn)
CompletableFuture<T> exceptionallyAsync(Function<Throwable, ? extends T> fn, Executor executor)
        
CompletableFuture<T> whenComplete(BiConsumer<? super T, ? super Throwable> action)
CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action)
CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action, Executor executor)
        
       
<U> CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn)
<U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn)
<U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor)
```

whenComplete 和 handle 的区别如果你看接受的参数函数式接口名称你也就能看出差别了，前者使用Comsumer, 自然也就不会有返回值；后者使用 Function，自然也就会有返回值

##### 异步 async

CompletableFuture 提供的所有回调方法都有两个异步（Async）变体，都像这样

```java
// thenApply() 的变体
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

