### MySQL什么时候会选错索引？

对应平常删除历史数据和新增数据的场景，就可能发生选错索引。

### 问题复现

```sql
# 创建表，主键id，索引a和索引b
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB；
```

```sql
# 插入(1,1,1)，(2,2,2)...(100000, 100000, 100000)十万条数据，分析一条SQL语句：
mysql> select * from t where a between 10000 and 20000;
```

上面的SQL肯定会走索引a，我们再看如下操作：

![img](https://static001.geekbang.org/resource/image/1e/1e/1e5ba1c2934d3b2c0d96b210a27e1a1e.png)

A开启一个事务，B删除了表t，然后调用idata()存储过程（其中就是插入了100000条数据），之后解释 SQL语句的执行过程。为了对比，对比强制使用索引a的查询语句：

```sql
set long_query_time=0;
select * from t where a between 10000 and 20000; /*Q1*/
select * from t force index(a) where a between 10000 and 20000;/*Q2*/
```

然后发现，Q1扫描了10万行，走了全表扫描，花费40ms；而Q2扫描10001行，执行21ms，**所以我们发现 没有使用 force index(a)强制走a索引的时候，MySQL选错了索引，或者说没有走索引。**

### 为什么？

## 1.**优化器估计扫描行数错误**

#### 优化器的逻辑

优化器选择最佳 执行策略 时，会考虑 **需要扫描的行数**、**是否使用零时表**、**是否排序** 等情况。

*对于上面的SQL语句来说，只需要预估扫描的行数，那么优化器是怎么预估扫描行数的？*

MySQL执行SQL前，不能知道满足条件的行数有多少，只能根据 **统计信息** 来估算，统计信息就是 **索引的区分度**，索引的区分度越大，可能需要扫描的行数就比较少，而索引的区分度是根据 索引的不同值的个数 确定的，**索引的不同值的个数** 叫做 **索引的基数**。

***索引的基数***

索引的基数是怎么确定的呢？

MySQL不可能统计所有的行，计算不同索引的个数，这样花费太昂贵。只能通过**采样**的方式估算。

采样统计时，innodb选择N个数据页，计算一个数据页上的平均的不同索引的个数，然后x数据页的个数，得到 索引的基数的采样估计值。![img](https://static001.geekbang.org/resource/image/16/d4/16dbf8124ad529fec0066950446079d4.png)

可以从上图看到，索引的基数值，虽然不同，但是差别不大，导致MySQL选错索引除了可能 索引基数统计错误(导致索引的区分度计算错误，导致估算的扫描行数错误，导致优化器选错索引。)，还有别的原因。

**下面是优化器选择的执行策略，看估计的扫描行数 rows**，Q1的rows为104620，而Q2走了index a的则是37116，最终innodb优化器选择了 Q1没有走索引，而是全表扫描，优化器为什么会觉得 Q1比Q2更优呢？

因为除了扫描行数，走index还需要回表，最终优化器觉得Q1更优。

**所以，这里优化器选错索引的原因在于，没有准确的判断出扫描行数。**

**使用analyze table t重新统计索引信息，解决优化器错误判断走索引需要扫描的行数的问题。**

![img](https://static001.geekbang.org/resource/image/e2/89/e2bc5f120858391d4accff05573e1289.png)

**mysql> analyze table t;** 重新统计索引信息。

![img](https://static001.geekbang.org/resource/image/20/9c/209e9d3514688a3bcabbb75e54e1e49c.png)

### 2.为了避免排序导致选错索引

比如下面的SQL语句，index a需要扫描1000条数据，而index b需要扫描50000条数据；然后都需要回表筛选b或者a的条件。

可以发现选择index a更快，但是 expain后发现 优化器选择的是 走index b。

这是因为优化器认为index b可以避免排序（就是说index b本身就是有序的，不需要再排序）。

```sql
mysql> select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 1;
```

![image-20210520212948297](F:\研究生\review-note\references-figures\image-20210520212948297.png)

### 怎么解决选错索引的问题？

使用explain查询执行策略，如果和预期不一致，选错了索引。可能是优化器分析需要扫描的行数出错，用analyze table t;重新统计索引统计。

也可以修改SQL语句，比如上面的SQL改为order by b,a ；由于limit 1，选择的结果也不变，但是需要按 b,a排序，这个时候优化器就不会认为b是优先级较高的索引，可以不排序，将a，b索引在排序上拥有一样的选择权重。

```sql
mysql> select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 1;
# 改为
mysql> select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b,a limit 1;
```

除此之外，还可以增加 更优的索引，或者可能可以直接删除走的错误索引（如果发现索引不是必要的）。

#### 开头 Session A 和 Session B 中的问题。

开头例子中，Session A开启一致性视图事务，一直未提交，Session B删除表t的所有数据，然后重新插入数据，最后执行查询。

发现Session B中的查询进行了全表扫描，而没有走索引。

这是因为：Session A开启重复读的一致性视图，需要在事务过程中读到事务开启时的数据，所以这里Session B中

的删除不会真正地删除 数据页中的数据，而是标记一下，后面的插入是直接在 原来的数据后插入，因此扫描预估的行数增大，为30000多行，而不是我们想的10001行。

*问题是，这里为什么不是使用 undo log + 当前数据进行回滚找到Session A的事务视图？而需要保存原来数据？*