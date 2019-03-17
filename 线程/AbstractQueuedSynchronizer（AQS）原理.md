# AbstractQueuedSynchronizer（AQS）原理

##### 一、概述

AbstractQueuedSynchronizer简称AQS，抽象的队列式的同步器。AQS定义了一套多线程访问共享资源的**同步器框架**。内部是**模板方法模式**。继承它、重写它的核心方法。主流的子类：ReentrantLock、Semaphore、CountDownLatch等。

##### 二、AQS框架

![3](C:\Users\YuChen_Xu\Desktop\工具手册\每日知识点\NOTE\images\3.PNG)

1、由上图可知，AQS框架的核心之一是维护了一个**volatile** int state（共享资源）变量和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）。

三种操作state的方式：

```java
getState()：获取state变量的值
setState()：设置state变量的值
compareAndSetState()：CAS算法设置state变量的值
```

2、AQS定义了两种资源共享方式：

​	①Exclusive：独占，只有一个线程可以获得锁，其他线程都必须等到拥有锁的线程释放掉锁后才可以争夺锁，如**ReentrantLock**。

​	②Share：共享，多个线程可同时获得锁，同时执行。如：**Semaphore、CountDownLatch**。

不同自定义同步器争用共享资源的方式也不同。但是**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可**。至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经帮我们在顶层实现好了，所以我们在定义同步器实现时只需要实现以下几种方法即可：

```java
isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要实现它。
tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)：独占方式。尝试释放资源，成功返回true，失败返回false。
tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余资源可用；正数表示成功，且有剩余资源。
tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待节点返回true，否则返回false。
```

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock时，会调用tryAcquire()独占锁并将state+1，以后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其他线程才有机会获取该锁。在释放锁之前，A线程自己本身可以重复获取此锁（state会累加），这就是**锁重入**的概念。注：获取多少次锁就要释放多少次锁，这样才能保证state是回到0的。 

再以CountDownLatch为例，任务分为N个子线程取执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是**并行**执行的，**每个子线程执行完后countDown()一次**，state会**CAS减1**。等到所有子线程都执行完后（即state=0），会**unpark()**主调用线程，然后主调用线程就会从await()函数返回，继续后面动作。 















