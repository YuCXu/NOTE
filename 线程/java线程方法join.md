# Thread.join()方法

##### 一、API

```java
public final void join() throws InterruptedException //等待该线程终止
```

抛出：`InterruptedException` - 如果任何线程中断了当前线程。当抛出该异常时，当前线程的中断状态被清除。

```java
public final void join(long millis) throws InterruptedException
//等待该线程终止的时间最长为 millis 毫秒。超时为 0 意味着要一直等下去。
```

参数：`millis` - 以毫秒为单位的等待时间。 

抛出：`InterruptedException` - 如果任何线程中断了当前线程。当抛出该异常时，当前线程的中断被清除。 

```java
public final void join(long millis,int nanos) throws InterruptedException
//等待该线程终止的时间最长为 millis 毫秒 + nanos 纳秒。
```

参数：`millis` - 以毫秒为单位的等待时间。`nanos` - 要等待的 0-999999 附加纳秒。

抛出：`InterruptedException` - 如果任何线程中断了当前线程。当抛出该异常时，当前线程的中断状态 被清除。

总结：Thread.join()，用来指定当前主线程等待其他线程执行完毕后，再来继续执行**Thread.join()后面**的代码。

##### 二、Demo

例1：

```java
import java.util.Date;
import java.util.concurrent.TimeUnit;
public class TestDemo1 implements Runnable {
	@Override
	public void run() {
		System.out.printf("begin: %s\n",new Date());
		try {
			TimeUnit.SECONDS.sleep(4);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.printf("finish:%s\n",new Date());
	}
	public static void main(String[] args) {
		TestDemo1 demo = new TestDemo1();
		Thread thread = new Thread(demo);
		thread.start();
		
		try {
			thread.join();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.printf("Main:%s\n",new Date());
	}
}
```

运行结果：

```java
begin: Thu Mar 14 21:14:32 CST 2019
finish:Thu Mar 14 21:14:36 CST 2019
Main:Thu Mar 14 21:14:36 CST 2019
```

如果去掉thread.join()，执行结果如下：

```java
Main:Thu Mar 14 21:18:23 CST 2019
begin: Thu Mar 14 21:18:23 CST 2019
finish:Thu Mar 14 21:18:27 CST 2019
```

例2：

```java
public class TestDemo1 implements Runnable {
	@Override
	public void run() {
		System.out.printf("begin: %s\n",new Date());
		try {
			TimeUnit.SECONDS.sleep(4);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.printf("finish:%s\n",new Date());
	}
	public static void main(String[] args) {
		TestDemo1 demo = new TestDemo1();
		Thread thread = new Thread(demo);
		thread.start();
		
		try {
			thread.join(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.printf("Main:%s\n",new Date());
	}
}
```

运行结果：

```java
begin: Thu Mar 14 21:21:51 CST 2019
Main:Thu Mar 14 21:21:54 CST 2019
finish:Thu Mar 14 21:21:55 CST 2019
```

thread.join(3000)：只要满足下面两个条件中一个时，主线程就会继续执行thread.join后的代码：

①Thread1执行完毕；

②已经等待thread执行了3000ms。

上面的例子中，thread执行了自身执行了4s，而设置的等待时间是3秒，因为thread等待的时间已经超时了，所以thread还没有执行完，主线程就开始执行后面的代码。

