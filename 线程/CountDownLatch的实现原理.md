# CountDownLatch的实现原理

##### 一、CountDownLatch的使用

CountDownLatch是同步工具类之一，可以指定一个**计数值**，在并发环境下由线程进行减1操作，当计数值变为0之后，被await方法阻塞的线程将会唤醒，实现线程间的同步。

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class TestCountDownLatch {
	public static void main(String[] args) {
		int threadNum = 10;
		final CountDownLatch countDownLatch = new CountDownLatch(threadNum);
		
		for(int i = 0 ; i < threadNum ; i++) {
			final int finalI = i + 1; 
			new Thread(()-> {
				System.out.println("thread"+finalI+"  start");
				Random random = new Random();
				try {
					Thread.sleep(random.nextInt(1000)+1000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println("thread"+finalI+"  finish");
				countDownLatch.countDown();
			}).start();;
		}
		try {
			countDownLatch.await();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println(threadNum +" thread all finish");
	}
}
```

运行结果：

```java
thread1  start
thread3  start
thread2  start
thread5  start
thread4  start
thread9  start
thread7  start
thread6  start
thread10  start
thread8  start
thread5  finish
thread7  finish
thread4  finish
thread6  finish
thread1  finish
thread3  finish
thread9  finish
thread10  finish
thread8  finish
thread2  finish
10 thread all finish
```

主线程启动10个子线程后阻塞在await方法，需要等子线程都执行完毕，主线程才能唤醒继续执行。 

##### 二、构造器

CountDownLatch和ReentrantLock一样，内部使用Sync继承AQS。构造函数很简单地传递计数值给Sync，并且设置了state。 

```java
Sync(int count) {
    setState(count);
}
```

AQS中的state，这是一个由子类决定含义的“状态”。对于ReentrantLock来说，state是线程获取锁的次数；对于CountDownLatch来说，则表示**计数值**的大小 

##### 三、阻塞线程

await方法：直接调用了AQS的acquireSharedInterruptibly。 

```jAVA
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

首先尝试获取共享锁，实现方式和独占锁类似，由CountDownLatch实现判断逻辑。 

```java
protected int tryAcquireShared(int acquires) {
   return (getState() == 0) ? 1 : -1;
}
```

返回1代表获取成功，返回-1代表获取失败。如果获取失败，需要调doAcquireSharedInterruptibly：

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

 doAcquireSharedInterruptibly的逻辑和独占功能的acquireQueued基本相同，阻塞线程的过程是一样的。不同之处： 

1、创建的Node是定义成共享的（Node.SHARED）；

2、被唤醒后重新尝试获取锁，不只设置自己为head，还需要通知其他等待的线程。（重点看后文释放操作里的setHeadAndPropagate）。

##### 四、释放操作

```java
public void countDown() {
    sync.releaseShared(1);
}
```

countDown操作实际就是释放锁的操作，每调用一次，计数值减少1： 

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

同样是首先尝试释放锁，具体实现在CountDownLatch中： 

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

死循环加上cas的方式保证state的减一操作，当计数值等于0，代表所有的子线程都执行完，被await阻塞的线程可以唤醒了，下一步调用doReleaseShared： 

```java
private void doReleaseShared() {
   for (;;) {
       Node h = head;
       if (h != null && h != tail) {
           int ws = h.waitStatus;
           if (ws == Node.SIGNAL) {
             //1
               if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                   continue;            // loop to recheck cases
               unparkSuccessor(h);
           }
           //2
           else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
               continue;                // loop on failed CAS
       }
       if (h == head)                   // loop if head changed
           break;
   }
}
```

标记1里，头节点状态如果SIGNAL，则状态重置为0，并调用unparkSuccessor唤醒下个节点。

标记2里，被唤醒的节点状态会重置成0，在下一次循环中被设置成PROPAGATE状态，代表状态要向后传播。

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

在唤醒线程的操作里，分成三步：

- 处理当前节点：非CANCELLED状态重置为0；
- 寻找下个节点：如果是CANCELLED状态，说明节点中途溜了，从队列尾开始寻找排在最前还在等着的节点；
- 唤醒：利用LockSupport.unpark唤醒下个节点里的线程。

线程是在doAcquireSharedInterruptibly里被阻塞的，唤醒后调用到setHeadAndPropagate。

 ```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
    
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
 ```

setHead设置头节点后，再判断一堆条件，取出下一个节点，如果也是共享类型，进行doReleaseShared释放操作。下个节点被唤醒后，重复上面的步骤，达到共享状态向后传播。

要注意，await操作看着好像是独占操作，但它可以在多个线程中调用。当计数值等于0的时候，调用await的线程都需要知道，所以使用共享锁。

#####  五、限定时间的await

CountDownLatch的await方法还有个限定阻塞时间的版本。

```java
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

最后来看doAcquireSharedNanos方法，和上文介绍的doAcquireShared逻辑基本一样，不同之处是加了time的处理。 

```java
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



 进入方法时，算出能够执行多久的deadline，然后在循环中判断时间。注意到代码中间有句： 

```java
nanosTimeout > spinForTimeoutThreshold
```

```java
static final long spinForTimeoutThreshold = 1000L;
```

spinForTimeoutThreshold写死了1000ns，这就是所谓的自旋操作。当超时在1000ns内，让线程在循环中自旋，否则阻塞线程。 

 

 

 

 

 

 

 