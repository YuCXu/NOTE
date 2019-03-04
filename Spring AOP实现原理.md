# Spring AOP实现原理

##### 代理模式：①静态代理：：需要为每个目标角色，创建一个对应的代理角色，类的数量会急剧膨胀 

##### 	            ②动态代理：自动为每个目标角色生成对应的代理角色 。

##### SpringAOP使用动态代理：

​		    ①JDK动态代理：只能对实现了接口的类产生代理

​		    ②Cglib动态代理：对**没有**实现接口的类产生代理对象。

##### 一、JDK动态代理：

核心接口（java.lang.reflect包里的InvocationHandler接口）：

```java
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

我们对于被代理类的操作都会由该接口中的invoke方法实现，其中的参数的含义分别是：

​	proxy：被代理的类的实例

​	method：调用被代理的类的方法

​	args：该方法需要的参数

静态方法（java.lang.reflect包中的Proxy类的newProxyInstance方法）：

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
throws IllegalArgumentException
```

 其中的参数的含义分别是：

​	loader：被代理的类的类加载器

​	interfaces：被代理类的接口数组

​	invocationHandler：调用处理器类的对象实例 

##### 实际例子：

Fruit接口：

```java
public interface Fruit {
    public void show();
}
```

Apple类实现Fruit接口：

```java
public class Apple implements Fruit{
    @Override
    public void show() {
        System.out.println("吃苹果");
    }
}
```

代理类

```java
public class MyInvoctionHandler implements InvocationHandler {
	Object object=null;//目标对象
	public MyInvoctionHandler(Object object) {
		super();
		this.object = object;
	}
	
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("洗苹果");
		Object obj=method.invoke(object, args);//obj为目标对象调用方法的返回值
		System.out.println("扔掉苹果核");
		return obj;
	}
}
```

编写测试用例：

```java
public class Test1 {
	@Test
	public void test() {
		Fruit fruit = new Apple();
		ClassLoader classLoader = fruit.getClass().getClassLoader();
		MyInvoctionHandler myInvoctionHandler = new MyInvoctionHandler(fruit);
		Fruit f = (Fruit) Proxy.newProxyInstance(classLoader, fruit.getClass().getInterfaces(), myInvoctionHandler);
		f.show();
	}
}
```

结果：

![1551705470895](C:\Users\YUCHEN~1\AppData\Local\Temp\1551705470895.png)

**二、CGlib动态代理：**

实现：

```java
public class CGlib implements MethodInterceptor {
    private Object proxy;
    public Object getInstance(Object proxy) {
        this.proxy = proxy;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.proxy.getClass());
        // 回调方法
        enhancer.setCallback(this);
        // 创建代理对象
        return enhancer.create();
    }
    //回调方法
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println(">>>>before invoking");
        //真正调用
        Object ret = methodProxy.invokeSuper(o, objects);
        System.out.println(">>>>after invoking");
        return ret;
    }
    
    public static void main(String[] args) {
        CGlib cGlib = new CGlib();
        Apple apple = (Apple) cGlib.getInstance(new Apple());
        apple.show();
    }
}
```







