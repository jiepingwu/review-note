### volatile

#### 前言

​	我们知道，volatile 变量规定了 happen-before规则。对一个 volatile 变量的写 happen-before 后续对这个变量的读。

#### volatile 可见性是怎么实现的？

​	JMM对于volatile变量实现了 happen-before规则，使用**内存屏障**来实现，当读写一个volatile变量时，所有该拥有该变量的缓存行失效，读写都将通过主存。有序性其实是使用内存屏障带来的一个性质，(也就是只有有序性保证了，才能保证可见性。)



#### 内存屏障(Memory Barries / Fences)

​	为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

1. ​	在每个volatile写操作前插入 StoreStore屏障
2. ​	在每个volatile写操作后插入 StoreLoad屏障
3.  在每个volatile读操作前后插入 LoadLoad 和 LoadStore屏障



volatile 的读写语义

当读写一个volatile变量的时候，JMM将线程对应的缓存行设置为无效，直接从主存中读写变量。

注意，高速缓存读写以缓存行为单位，所以可能会产生 缓存行的伪共享 导致 效率降低。