### happen before 原则

#### 写在前面

我们知道，多线程环境中存在三大问题：**可见性，有序性，原子性**。

可见性：为了解决CPU运算速度快而内存IO速度相比太慢的问题，现代CPU增加了多级缓存，CPU只和缓存进行读写，提高IO效率。但是这样带来了 可见性的问题，因为不同线程使用的缓存中的数据不一致。

有序性：为了加快CPU执行指令的速度，CPU进行汇编指令的优化，允许进行指令重排序，这带来了 有序性 的问题，即生成的字节码最后编译为汇编指令时可能不是按照原来的顺序执行的，在不改变结果的前提下，优化指令的执行顺序，比如：将执行速度快的指令放到前面。

原子性：为了保证一个操作是不可被拆分的整体，就是说不能有其他的线程在这个原子操作过程中来插入一个操作。如果一些操作不满足原子性，我们就可能拿到操作到一半的数据。



JVM就规定了一系列的happen before原则来解决可见性的问题。

- [x] #### happen before

happen before规定了如果两个操作具有happen-before关系，那么前一个操作的结果必须对后一个操作及时可见。

一个例子来看看happen before。

```java
class ReorderExample {
    int x = 0;
    boolean flag = false;
    public void write() {
        x = 42;				// 1
        flag = true;		// 2
    }
    public void read() {
        if (flag)			// 3 
            System.out.println(x); // 4
    }
}
```

这里A线程执行write()，B线程执行read()。那么B可以读到的x的值为0，而不是42。因为 语句2和语句1 可能发生 指令重排序，导致flag=true时，x=0。解决的方式是：将 flag 改为 volatile 类型，使其对于其他线程及时可见。

##### happen before 6个规则

happen before有很多规则，用于保证我们线程的可见性安全。这里列举6个常见的规则。

##### 1. 程序顺序性原则

​	一个线程中的每个操作 happen before 该线程中的任意后续操作。（注意这里是单线程）

##### 2. volatile变量的读写原则

​	对一个volatile变量的写 happen before 任意后续对这个volatile的读。

​	非volatile变量的读写 happen before volatile变量的读写。

​	*非volatile read write ---> volatile write ---> volatile read*

volatile的happen-before原则的实现，是通过 **内存屏障来实现**。

##### 3. 传递性原则

​	A happen-before B, B happen-before C  则有 A happen-before C

##### 4. synchronized 锁原则

​	对一个锁的解锁 happen-before 随后对这个锁的加锁。

​	也就是说，synchronized代码块中的变量，在锁释放后，对于其他获得该锁的线程是及时可见的。

##### 5. start() 原则

​	如果A线程执行 ThreadB.start() (启动B线程)，那么A线程的ThreadB.start()操作 happen-before 于线程B中的任意操作。

​	也就是说，A线程中启动了B线程后，B线程可以看到 A线程启动B线程前的所有操作。

##### 6. join 原则

​	如果线程A执行 ThreadB.join()并成功返回，那么线程B中的任意操作 happen-before 于线程A。

​	也就是说，A线程中调用线程B的join后，线程B的操作对于线程A及时可见。



##### 总结

1. happen-before 规定了前一个操作对于后一个操作的及时可见性，同时带来了有序性。
2. start 和 join 也是解决主线程和子线程通信的方式之一。
3. 从内存语义上来讲，volatile 变量的 read-write 和 锁的release-acquire 有相同的内存语义，也就是说，在只解决可见性的前提下，synchronized 和 volatile 有相同的作用。**注意，volatile不会带来原子性，或者说只保证了 一个read-write赋值操作的原子性。**而synchronized锁由于保证了线程互斥，所以整个同步代码块中同一时刻只有一个线程执行，进而保证了代码块的原子性。