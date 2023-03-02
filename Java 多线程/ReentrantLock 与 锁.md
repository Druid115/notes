## ReentrantLock 与 锁

### ReentrantLock 与 synchronized 区别

- 可重入性

- 锁的实现：synchronized 依赖于 JVM 实现的，ReentrantLock 是 JDK 实现的

- 性能区别，现在都是在用户态把加锁的问题解决，避免线程进入内核中的阻塞状态，基本无差别

- 功能区别：

  ReentrantLock 独有的功能：

  - 可指定是公平锁（先等待的线程先获得锁）还是非公平锁，synchronized 只能是非公平锁；
  - 提供了一个 Condition 类，可以分组唤醒需要唤醒的线程；
  - 提供能够中断等待锁的线程的机制，lock.lockInterruptibly()。

