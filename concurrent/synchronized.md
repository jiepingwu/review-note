### Synchronized

​	synchronized是jvm内置的互斥锁，提供了可见性和原子性。

##### 实现上

​	在实现上，synchronized分为同步代码块和同步方法，同步代码块在进入和离开临界区的时候，加上了 **monitorenter 和 monitorexit**字节码指令，monitorenter 请求锁，如果没有申请到锁，则加入监视器对象的等待队列中。monitorexit释放锁，如果监视器对象的锁状态为0则释放锁，唤醒等待队列中的线程争抢锁，如果大于0的话代表锁重入，状态量减一。而同步方法则在方法上加上了synchronized标志。

##### 缺点

​	**synchronized内置锁是 互斥的公平锁。** 没有可中断获取锁的方法，不会抛出异常，不能延时获取锁，获取锁一定会被阻塞。

​	这也是Doul L加入了AQS和ReentrantLock的原因。

另外，为了提高效率，jvm对于 synchronized 进行了优化，见*锁升级过程*。