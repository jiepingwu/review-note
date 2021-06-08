### order by 是怎么工作的？

这一节我们看看 order by 语句的执行过程，以及什么样的参数会影响执行的行为。

假设一个场景，一个 市民表，我要查询城市是"杭州"的所有人的名字，并且按照姓名排序返回前 1000 个人的姓名，年龄。

假设表定义如下：

```sql
CREATE TABLE `t` (
	`id` int(11) NOT NULL,
    `city` varchar(6) NOT NULL,
    `name` varchar(6) NOT NULL,
    `age` int(11) NOT NULL,
    `addr` varchar(128) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `city` (`city`)
) ENGINE=InnoDB;
```

这个时候，你的 SQL 语句可以这么写：

```SQL
select city, name, age from t where city='杭州' order by name limit 1000;
```

我们下面就来看看 这样一条语句的执行过程。

#### 全字段排序

​	我们了解了索引，所以一般我们为了避免全表扫描，会将 city 字段加上 索引，然后使用 explain 命令来看看这个语句的执行计划。

![img](https://static001.geekbang.org/resource/image/82/03/826579b63225def812330ef6c344a303.png)

​	可以看到，Extra 字段中的 Using index condition 表示走了index，Using filesort 表示 需要排序。

##### sort_buffer

​	MySQL 会在需要排序的时候 分配一块内存用于排序，也就是 sort_buffer。

​	我们假设索引结构如下：

![image-20210606203826243](..\references-figures\image-20210606203826243.png)

​	一般来说，使用一个 index 字段用于排序的时候，执行过程如下：

1. 首先根据 index，查找 index tree，找到对应条件的索引(也就是这里的city = '杭州')的主键ID，根据主键ID回表。
2. 回表根据 主键ID 取出整行数据，取其中的需要字段(也就是这里的 name, city, age字段的值)，将这些值加入 **sort_buffer**。
3. 重复步骤2，直到索引 city 的值不满足条件。
4. 在 sort_buffer 中，根据 排序字段 name 做快速排序。
5. 将结果取需要的值（limit 1000，前1000行返回给客户端）。

我们暂时把上面这个过程叫做：**全字段排序**

其中，**如果 sort_buffer 不够，那么则会使用 磁盘临时文件 辅助排序，sort_buffer_size是可以设置buffer大小的参数。**

可以使用下面的语句，来确定一个排序语句是否使用了临时文件外部排序。

```sql

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

![img](https://static001.geekbang.org/resource/image/89/95/89baf99cdeefe90a22370e1d6f5e6495.png)

其中 number_of_tmp_files 表示的是，排序过程中使用的临时文件数。为什么要使用这么多文件呢？

因为 **MySQL 排序过程中，如果sort_buffer内存不够用，则就会使用 磁盘临时文件 进行外部排序，而 外部排序 一般都使用 归并排序，也就是说，MySQL 会将 需要排序的数据 分成多份(比如这里的12份)，然后对于每个文件多线程分别进行排序，最终再将这些文件合并为一个有序的大文件。**



#### rowid 排序

​	对于上面的排序，只对原表的数据读了一遍，**排序的操作都在 sort_buffer 或者 磁盘临时文件 中进行**，但是，**对于 需要查询的字段很多 的情况，就需要将这些字段 放入 sort_buffer 内存中**，这样 sort_buffer 能放下的数据行数就很少，很容易 sort_buffer 不足，导致使用 磁盘临时文件 进行归并排序，**排序的性能会很差**。

​	所以，上面的排序才被称为 "全字段排序" （再次强调：这里的全字段含义是，全部需要返回的字段都会被 放入 sort_buffer 中，如果使用内存排序的话。）

​	所以，对于单行很大，也就是说需要返回的字段数量很多的情况，上面那种 全字段排序 的效率不够好。

那么，对于单行很大，**MySQL 有什么优化？或者说，MySQL会怎么做？**

修改 max_length_for_sort_data 参数，让 MySQL 采用另一种算法。

```Mysql
SET max_length_for_sort_data = 16;
```

**max_length_for_sort_data 参数**

​	max_length_for_sort_data 参数是 MySQL 中控制用于排序的行数据的长度的一个参数，也就是说，如果 单行长度超过该值，MySQL 就认为行数据太长，就会使用另一个算法，而不是使用 全字段排序。



那么这个时候，MySQL 排序的执行过程是怎么样的？

1. 同样的，从索引树找到对应的主键ID，返回主键索引树。
2. 区别在于，从主键索引树上取出的数据不再是全部需要返回的字段，而只是 **排序字段name 和 主键 row id**，将这两个值存入 sort_buffer 中。
3. 继续下一条 索引记录，重复上面的2步骤，将 排序字段name 和主键ID 存入 sort_buffer 中。
4. 对 sort_buffer 中的数据按照 排序字段 name 进行排序。
5. 然后，**遍历排序结果，取limit 1000 行数据，并且根据这些数据的 id 值回表，取出剩余的 city, name, age 三个字段，返回客户端。**

简单来说，这种排序方法，不会将需要返回的全部字段都加载进内存中进行排序，而是 只将 排序字段 和 row id 放入sort_buffer中，然后排序完成后，再根据 row id 回表，查到需要的剩余全部字段 返回给 客户端。



#### 全字段排序 VS row id 排序

总结：

1. 如果 MySQL 认为 排序内存太小，行数据长度超过参数 max_length_for_sort_data，放不下需要返回的字段，那么就会采用row id 排序，这种方法只会将 排序字段 和 row id 放入 sort_buffer 中排序，排序完成后再根据row id 会表查询 剩下需要的全部字段。
2. 如果 MySQL 认为 排序内存够大，行数据长度没有超过 max_length_for_sort_data，则会使用 全字段排序，这种方法将 需要返回的全部字段都 加载进 sort_buffer 内存中，而如果此时内存不足，则会使用 外部的磁盘临时文件 进行归并排序。这种方法 不需要回表，直接可以从内存中返回。

这里显示了MySQL的一个设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问。

对比：

1. 全字段排序在内存足够的情况下 效率是比较好的，因为它相比于 row id 排序，不需要回表，减少了磁盘IO。
2. 对于 行数据长度过大，就会导致 sort_buffer 不足，然后使用 磁盘临时文件进行外部归并排序，这个效率是比较低的。



**有没有更好的方式优化？**

对于我们一开始的案例，有没有更好地方式，加快 order by 语句的速度？

答案是有的，我们想一想，**如果我们一开始 从 city index tree 上查到的数据就是 对于 name 有序的，那么不就不用再排序了**吗？

​	没错，我们可以使用 **联合索引 (city, name) **，使用下面语句：

```sql
alter table t add index city_user(city, name);
```

于是，这样我们的 order by 查询，就不需要排序，不需要临时表了。

![img](https://static001.geekbang.org/resource/image/fc/8a/fc53de303811ba3c46d344595743358a.png)



**还能不能优化？**

对于上面的查询，我们还能不能优化？没错，使用索引覆盖，减少回表操作。

那么，我们就可以 创建一个 city, name, age 的联合索引。

```sql
alter table t add index city_user_age(city, name, age);
```

这个时候，对于 city 相同的行来说，其 name 也是有序递增的，此时的 order by 也不需要排序，并且由于索引覆盖，不需要回表。执行过程如下：

1. 从 (city, name, age) index tree 中找到满足条件 city = '杭州' 的数据，直接将其中的 city, name, age 加入结果集。
2. 直到 查询到满足条件的数据 有 1000 条，或者剩余没有满足条件的数据。

当然，加索引，维护索引是要 成本的，并不是越加越好的，需要权衡决定。
