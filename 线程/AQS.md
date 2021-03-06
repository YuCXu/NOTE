# AQS

AQS（AbstractQueuedSynchronizer），这个类在java.util.concurrent.locks下。

##### 1.1 AQS原理概括

AQS的核心思想：如果被请求的共享资源空闲，则将当前请求资源的线程**设置为有效的线程**，并且将共享资源设置为锁定。如果被请求的共享资源被占用，那么就需要一套**线程阻塞等待以及被唤醒时锁分配**的机制，这个机制AQS用的时**CLH队列**实现的，即将暂时获取不到锁的线程加入到队列中。

​	注：CLH队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS是将每条请求共享资源的线程封装到一个CLH锁队列的一个结点（Node）来实现锁的分配。

##### 1.2 AQS对资源共享的方式

①Exclusive（独占）：只有一个线程能执行，如ReentranLock。又可分为公平锁和非公平锁。

​	公平锁：按照线程在队列中的排队顺序，先到者先拿到锁。

​	非公平锁：无视队列顺序直接去抢锁，谁抢到就是谁的。

②share（共享）：多个线程可以同时执行。

##### 1.3 AQS底层使用了模板方法模式

自定义同步器时需要重写下面几个AQS提供的模板方法：

```java
tryAcquire(int);//独占方式。尝试获取资源
tryRelease(int);//独占方式。尝试释放资源
tryAcquireShared(int);//共享方式。尝试获取资源。
tryReleaseShared(int);//共享方式。尝试释放资源
```

CountDownLatch：任务分为n个子线程去执行，state也初始化为N（N与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown（）一次，state会CAS减一。等到所有子线程都执行完后（即state=0），会unpark（）主调用线程，然后主调用线程就会从await（）函数返回，继续后续工作。