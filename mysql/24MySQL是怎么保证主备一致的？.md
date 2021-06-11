![img](https://static001.geekbang.org/resource/image/a6/a3/a66c154c1bc51e071dd2cc8c1d6ca6a3.png)

​	**bin log 不仅可以用来归档，还可以用来做 主备同步。**是怎么做的？为什么备考执行了 bin log 就可以跟 主库 保持一致？

### MySQL 主备一致的原理

简单看这个图，就是主备切换流程，Client 先是访问 节点A，而B是A的备库，只是将 A 的更新都同步过来，到本地执行，这样保证 节点 B 和 节点 A 的数据是一致的。

![img](https://static001.geekbang.org/resource/image/fd/10/fd75a2b37ae6ca709b7f16fe060c2c10.png)

当需要切换的时候，就切换成状态2，这时候 Client 访问读写的都是 节点B，而节点A是B的备库。

而我们通常建议将 备考设置为 readonly 模式，这样会有以下好处：

- 有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作；
- 防止切换逻辑有 bug，比如切换过程中出现双写，造成主备不一致；
- 可以用 readonly 状态，来判断节点的角色。

那备库设置为 readonly ，怎么和主库保持同步一致呢？

readonly 设置对于 super 权限用户是无效的，而用于同步更新的线程，就拥有 super 权限。



**节点A到节点B这条线的内部流程是怎样的，也就是 主备如何同步？**

![img](https://static001.geekbang.org/resource/image/a6/a3/a66c154c1bc51e071dd2cc8c1d6ca6a3.png)

如上图，图中还包括了 **bin log 和 redo log 的双阶段写过程**，主库接受Client请求后，执行**内部事务逻辑**，同时**写 bin log** 。

而 **备库B 和 主库A 之间维持了 一个长连接，主库A 内部有一个线程，专门用于服务备库B的这个长连接**。

一个**事务日志完整的同步过程**是这样：

1. 在备库B 上 通过 change master 命令，设置 主库A 的IP、端口、用户名、密码、**以及从哪个位子开始请求 bin log ，也就是同步位置，这个位置 包括 文件名 和 日志偏移量。**
2. 在备库B 上执行 start slave 命令，**这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread，其中 io_thread 负责与主库连接连接。**
3. 主库 A 校验完 用户名、密码后，开始按照 备库 B 传过来的位置，从本地读取 bin log，发给 B。
4. 备库 B 拿到 bin log 后，写入到 本地文件，称为 **中转日志(relay log)。**
5. sql_thread 读取中转日志，解析其中的命令，并且执行。

需要说明的是：由于多线程复制方案的引入，sql_thread 变为了 多个线程。



**bin log 里面到底有什么内容？为什么可以执行被备库执行？**

**bin log 的三种格式比较**

1. statement

    记录 SQL 语句的原文，这个从命名就可以发现。

    可能会导致 主备操作不一致的情况，原因在于：主备走的索引可能不一样，导致操作不一致。

2. row

    记录真实操作行的主键ID，这样当备库使用bin log的时候，就不会出现主备操作不一致的情况。

3. mixed

    为什么会有 mixed 格式的 binlog？

    我们先考虑 statement 和 row 的弊端。statement可能会导致主备不一致，所以要用 row ，但是 row 很占空间，比如一个delete 10万行数据的语句，statement只需要保存一条 sql 语句，而 row 则需要把这10万条记录都写到 bin log 中，占用空间的同时也会消耗更多的IO资源。

    所以 才有了这种 mixed 模式：**mixed模式下，MySQL会判断这条语句是否会导致主备不一致，如果可能，则使用 row ，否则使用 statement 模式。**

使用选择，选择上，应该不使用 statement，至少应该使用 mixed 模式，而现在越来越多的场景会使用 row 模式。

row 模式还有最大的一点好处：**恢复数据**

row 会记录删除语句执行后删除的结果，可以用来恢复，

记录 insert 语句中 所有字段的信息，可以定位到刚刚被插入的那一行、

记录 update 语句下 更新前和更新后的整行数据，可以用于恢复。



**循环复制问题**

​	上面我们了解了 主备一致 的实现，bin log 的特性保证了 在备库执行 bin log ,可以得到与主库相同的状态。

​	所以，我们可以认为正常情况下，主备是数据一致的，也就是图中的A,B两个节点是数据一致的。

​	上面我们说的其实是 M-S 结构，也就是 一个 Master 一个 Slaver，但是 实际生产 中比较多的是 双Master结构。

​	也就是 大家都是 Master ，大家也都是 Slaver。如下图：

![img](https://static001.geekbang.org/resource/image/20/56/20ad4e163115198dc6cf372d5116c956.png)

**那这样就会带来一个问题：A是Master，发送 bin log 给 B，B也是 Master，又发送 bin log 给A，那这样不是 循环一直 执行 bin log。**

​	怎么解决的？

​	MySQL 在这个 binlog 中记录了这个命令第一次执行时所在实例的 server id。

1. 规定两个库的 server id 必须不同，如果相同，不能为主备关系。
2. 一个备库接受到bin log 并重放的过程中，生成与原 bin log 的 server id 相同的 bin log。
3. 每个库在接受到 Master 发来的 bin log 后，先判断 server id ，如果和自己的相同，表示这个 日志时自己生成的，就丢弃。

**简单来说，就是 bin log 在第一次执行时，保存执行的库的 server id，如果后续执行 bin log 的库和这个 server id 相同，就直接丢弃，阻止循环。**

