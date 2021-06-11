### MySQL是怎么保证数据不丢的？

前面我们知道，只要 redo log 和 bin log 保证持久化到 磁盘，就能确保MySQL 异常重启后，数据可以恢复。

那么这篇我们就来看看，redo log 和 bin log 是怎么写入 磁盘的。

##### bin log 的写入机制

​	bin log 的写入逻辑很简单：

事务执行过程中，先把日志写入到 bin log cache，事务提交的时候，再把 bin log cache 写到 bin log 文件中。

系统给 bin log cache 分配了 一片内存，每个线程一个，参数 binlog_cache_size 用于控制单个线程内 bin log cache 所占内存的大小，如果超过这个参数，则要暂存到磁盘。

事务提交的时候，执行器把 bin log cache 里面完整事务写入到 bin log 中，并清空 bin log cache。

##### redo log 的写入机制

​	前面我们知道了，redo log 是会先写到 redo log buffer 中的。

而如果 MySQL 在事务执行过程中 异常重启，这部分的log就丢失了，由于事务没有提交，所以不会有什么损失。

另一个问题是：事务还没有提交的时候，redo log buffer 中的部分日志有没有可能会被持久化到磁盘？

答案是：确实会有。

这个问题，我们需要了解 redo log buffer 中的数据是怎样被写入到 redo log 中的。

redo log  可以会存在 3 个状态，如下图中3种颜色。

![img](https://static001.geekbang.org/resource/image/9d/d4/9d057f61d3962407f413deebc80526d4.png)

1. 存在 redo log buffer 中，物理上是在 MySQL 进程内存中，红色部分。
2. 写到磁盘(write)，但是还没有持久化(fsync)。物理上是在文件系统的 page cache 里，黄色部分。
3. 持久化到磁盘，对应 hard dist，绿色部分。

log写到 redo log buffer 和 write 到 page cache 的速度是很快的(也就是上面的红色和黄色部分)，但是绿色部分 持久化到磁盘是比较慢的。

为了控制 redo log 的写入策略，InnoDB 提供了 innodb_flush_log_at_trx_commit 参数：

1. 为0：每次事务提交 只是把 redo log 留在 redo log buffer 中。
2. 为1：每次事务提交 都直接将 redo log 持久化到磁盘。
3. 为2：每次事务提交 只是把 redo log 写到 page cache中。

**InnoDB 后台进程，每隔 1 s，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后 调用 fsync 持久化到磁盘。**

注意：事务执行过程中的 redo log 也是直接写到 redo log buffer 中，这些 redo log 也会被后台线程一起持久化到 磁盘，也就是说，**一个还没有提交的事务的 redo log ，也是可能已经被持久化到磁盘的。**

实际上，除了 后台线程每1s执行一次 刷新 redo log buffer 到磁盘中，还有 以下两种情况会 让一个没有提交 的事务 的 redo log 写入磁盘中：

1. **redo log buffer 占用控件即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。**
2. **并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。**

