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

自定义同步器要么是独占方法，要么是共享方法，它们也只需要实现tryAcquire—tryRelease、tryAcquireShared—tryReleaseShared中的一种即可。但AQS支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

##### 三、独占获取锁源码分析

**1、acquire（int）**

1.1 讲解

此方法是独占模式下线程获取共享资源的顶级入口。如果获取到资源，线程直接返回，若没有获取到资源（可能已经有线程占上了，别的线程无法再次进入），则将其放到等待队列的队列，直到获取到资源为止。整个过程忽略中断的影响。就是说你中断，我这不会中断，我只知道我欠你一次中断，所以我会等到流程结束后自行调用中断方法进行自我中断。

1.2 源码

```java
public final void acquire(int arg) {
   /**
    * tryAcquire()尝试获取资源，true：获取到了。false：没获取到。
    * addWaiter()将正在请求的线程放到队尾。
    * acquireQueued()进入等待队列，直到获取到资源为止才释放。
    * selfInterrupt()自我中断。
    */
   if (!tryAcquire(arg) &&
       acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
       selfInterrupt();
}
```

1.3 流程

①tryAcquire（）尝试直接去获取资源，若成功（第一把锁或者锁重入），则直接返回，不做其他任何额外处理。

②若没有获取到资源，调用addWaiter（）将该线程加入等待队列队尾，并标记为独占模式

③然后执行acquireQueued（）是线程在等待队列中获取资源，一直获取到资源后才返回。若整个过程被中断了，则返回true，否则返回false。（这里的返回是为了下面自我中断用）。

④若上面返回true，则需要自我中断，调用selfInterrupt（），将中断补上。

**2、tryAcquire（int）**

2.1 讲解

尝试去获取独占资源。若成功，则返回true，否则返回false。

2.2 源码

```java
/**
*这是模板模式，有子类重写了逻辑
*/
protected boolean tryAcquire(int arg) {
   throw new UnsupportedOperationException();
}
```

AQS只是一个框架，模板设计模式，具体的资源获取/释放都交给自定义同步器去实现了，因为每个人的实现方式和实现逻辑不同，所以AQS无法写。

**3、addWaiter（Node）**

3.1 讲解

此方法用于将当前线程添加到FIFO队列的队尾，并返回当前线程所在的节点。

3.2 源码

```java
private Node addWaiter(Node mode) {
   // 以给定模式（EXCLUSIVE独占和SHARED共享）构造节点。
   Node node = new Node(Thread.currentThread(), mode);
   // 尝试快速的方式直接放到队尾。
   Node pred = tail;
   // 若现有队列的队尾不为null
   if (pred != null) {
       // 则让当前线程节点的上一个节点指向队尾。也就是当前线程节点在队尾下方。
       node.prev = pred;
       // 通过CAS算法将队尾设置为当前线程的节点。
       if (compareAndSetTail(pred, node)) {
           // 队尾的下一个节点指向当前线程的节点。（双向队列，互相指向。）
           pred.next = node;
           // 返回节点
           return node;
       }
   }
   // 若上一步失败，则通过enq入队尾
   enq(node);
   return node;
}
```

3.3 流程

Node类是对每一个访问同步代码的线程的封装，其包含了需要同步的线程本身以及线程的状态，如是否被阻塞，是否等待唤醒，是否已经唤醒，是否已经被取消等等。变量waitStatus则表示当前线程被封装成Node结点的等待状态，共有四种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE。

①CANCELLED：值为1，在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消该Node的节点。其节点的waitStatus为CANCELLED，即结束状态，进入状态后的节点将不会再变化。

②SIGNAL：值为-1，被标识为该等待唤醒状态后继结点，，当其前结点的线程释放了同步锁或被取消，将会通知该后继节点的线程执行，就是：处于唤醒状态，只要前继结点释放，就会通知标识为SIGNAL状态的后继节点的线程执行。

③CONDITION：值为-2，与Condition相关，该标识的结点处于等待队列中。结点的线程等待在Condition上，在其他线程调用了Condition的signal（）方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。

④PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。

⑤0状态：值为0，代表初始化状态。

**AQS在判断状态时，通过用waitStatus>0表示表示取消状态，而waitStatus<0表示有效状态。**

**4、enq（Node）**

4.1 讲解

此方法用于将node加入队尾

4.2 源码

```java
private Node enq(final Node node) {
   // CAS自旋，直到加入队尾成功。
   for (;;) {
       Node t = tail;
       // 若队尾为null，则CAS创建新的节点作为队尾队首
       if (t == null) {
           if (compareAndSetHead(new Node()))
               tail = head;
       } 
       // 若队尾不为null，则将当前节点的上一个节点指向队尾，队尾通过CAS算法设置成当前节点。（双向队列，互相指定）
       else {
           node.prev = t;
           if (compareAndSetTail(t, node)) {
               t.next = node;
               return t;
           }
       }
   }
}
```

4.3 流程

CAS自旋volatile变量。

**5、acquireQueued(Node, int)**

5.1 讲解

进入等待状态开始休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源，然后就可以干自己想干的事。

5.2 源码

```java
final boolean acquireQueued(final Node node, int arg) {
   // 标记是否成功拿到资源。true：没拿到；false：拿到了。
   boolean failed = true;
   try {
       // 标记等待过程中是否被中断过。默认没中断过。
       boolean interrupted = false;
       // 又是一个经典的CAS自旋，不再解释。
       for (;;) {
           // 拿到当前节点的前驱（上一个）节点
           final Node p = node.predecessor();
           // 若前驱节点是head，那明显自己是第二个节点，那么变有资格（可能是老大释放完资源唤醒自己的也可能是被interrupt了）去尝试获取资源。
           if (p == head && tryAcquire(arg)) {
               // 拿到资源后，将head指向该节点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
               setHead(node);
               // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
               p.next = null; 
               // false获取资源成功的标识。
               failed = false;
               // 返回中断状态。起始调用者需要根据此状态判断是否需要自我中断
               return interrupted;
           }
           // 若不是第二个节点或获取资源失败，那么久进入waiting状态去休息，直到被unpark()唤醒
           if (shouldParkAfterFailedAcquire(p, node) &&
               parkAndCheckInterrupt())
               // 如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
               interrupted = true;
       }
   } finally {
       if (failed)
           cancelAcquire(node);
   }
}
```

5.3 流程

其中调用了两个方法：

```java
shouldParkAfterFailedAcquire();
parkAndCheckInterrupt();
```

##### 6、shouldParkAfterFailedAcquire(Node, Node)

6.1 讲解

主要用于检查状态，看看自己是否真的可以去休息了（进如waiting状态）。以防万一的操作。

6.2 源码

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
   // 拿到前驱状态
   int ws = pred.waitStatus;
   // 若前驱状态是-1，则可以安心休息了。相当于若已经告诉前驱吃完饭告诉我下，我去拿号（前驱是100%可信任的人）。
   if (ws == Node.SIGNAL)
       return true;
   if (ws > 0) {
       /*
        * 若前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在他的后面。
        * 注意：那些放弃的节点，由于被自己“加塞”到他们前面，他们相当于形成了一个无引用链，稍后就会被GC的。
        */
       do {
           node.prev = pred = pred.prev;
       } while (pred.waitStatus > 0);
       pred.next = node;
   } else {
       //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
       compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
   }
   return false;
}
```

6.3 流程

整个流程中，如果前继结点的状态不是SIGNAL，那么自己就不能安心去休息，需要找个安心的休息点，同时可以再尝试下有没有机会轮到自己拿号。

7、parkAndCheckInterrupt()

7.1 讲解

如果线程找好安全休息点后，那就可以安心去休息了。此方法就是让线程去休息，真正进入等待状态。

7.2 源码

```java
private final boolean parkAndCheckInterrupt() {
   // 调用park()使线程进入waiting状态
   LockSupport.park(this);
   // 如果被唤醒，查看自己是不是被中断的。
   return Thread.interrupted();
}
```

7.3 流程

park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：①被unpark（）；②被interrupt（）。需要注意的是，Thread.interrupted（）会清除当前线程的中断标记位。

##### 8、再谈5.3流程

①结点进入队尾后，检查状态，找到安全休息点；

②调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；

③被唤醒后，看自己是不是有资格拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程①。

##### 9、小结

9.1 acquire()源码

```java
public final void acquire(int arg) {
   /**
    * tryAcquire()尝试获取资源，true：获取到了。false：没获取到。
    * addWaiter()将正在请求的线程放到队尾。
    * acquireQueued()进入等待队列，直到获取到资源为止才释放。
    * selfInterrupt()自我中断。
    */
   if (!tryAcquire(arg) &&
       acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
       selfInterrupt();
}
```

9.2 总结流程

①调用自定义同步器的tryAcquire)尝试直接去获取资源，若成功则直接返回；

②没成功，则addWaiter()将该线程加入等待队列的尾部，并标记位独占模式；

③acquireQueued()是线程再等待队列中休息，有机会时（轮到自己，会被unpark()唤醒）会去尝试获取资源。获取资源后才返回，若在整个流程中被打断，则返回true，否则返回false；

④若线程在等待过程中被中断过，它是不响应的。它只是获取资源后才再进行自我中断selfInterrupt() ，将中断补上。

![4](C:\Users\YuChen_Xu\Desktop\工具手册\每日知识点\NOTE\images\4.PNG)









