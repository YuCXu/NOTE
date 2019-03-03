# ArrayList源码分析

- ArrayList是基于数组实现的，是一个动态数组，其容量能自动增长。

- ArrayList不是线程安全的，只能用在单线程环境下，多线程环境下可以考虑concurrent并发包下的CopyOnWriteArrayList类。

- 定义：①ArrayList<E>可以看出它是支持泛型的，它继承自AbstractList，实现了List、RandomAccess、Cloneable、Java.io.Serializable接口。 ②AbstractList提供了List接口的默认实现（个别方法为抽象方法）。 ③List接口定义了列表必须实现的方法。 ④RandomAccess是一个标记接口，接口内没有定义任何内容,支持快速随机访问，实际上就是通过下标序号进行快速访问。 ⑤实现了Cloneable接口的类，可以调用Object.clone方法返回该对象的浅拷贝。 ⑥通过实现 java.io.Serializable 接口以启用其序列化功能。未实现此接口的类将无法使其任何状态序列化或反序列化。序列化接口没有方法或字段，仅用于标识可序列化的语义。

  ![img](https://img.mubu.com/document_image/d4120c15-fd69-4e11-acb8-5e035b377be3-2159757.jpg)

- 属性：

  ![img](https://img.mubu.com/document_image/664f60a3-6b08-48e9-8487-7f2ca8654b2a-2159757.jpg)

  - ArrayList提供了2个私有属性，elementData和size。elementData存储ArrayList内的元素，size表示它包含的元素的数量。（实际数量）
  - transient关键字的作用：在采用Java默认的序列化机制的时候，被该关键字修饰的属性不会被序列化。

- 构造方法：ArrayList提供了三个构造方法

  ![img](https://img.mubu.com/document_image/de99a556-d833-4d0d-9549-457d9135ae4a-2159757.jpg)

  - 第一个构造方法使用initialCapacity来初始化elementData的数组大小；
  - 第二个构造方法创建一个空的容量为10的elementData的数组；
  - 第三个构造方法提供将集合转成数组并给elementData（返回若不是Object[]，则转换成Object[]）。

- 其他方法：

  - 元素存储：关于ArrayList元素的存储，提供了5个方法：

    ![img](https://img.mubu.com/document_image/b4958cef-cd68-4cab-b095-89fa77b923cb-2159757.jpg)

    - set方法：替换元素

      ![img](https://img.mubu.com/document_image/aa7a69c9-cd30-400d-abd1-bda889dec5d7-2159757.jpg)

    - add(E e)方法：尾部添加元素

      ![img](https://img.mubu.com/document_image/6bdaad18-3fa9-4fb0-908f-8aeb04b5e72c-2159757.jpg)

      ![img](https://img.mubu.com/document_image/186afa6e-7ee1-424f-8677-869850ede74a-2159757.jpg)

    - add(int index, E element)方法，指定位置添加元素

      ![img](https://img.mubu.com/document_image/26977c81-dcd3-46a7-9e10-a50c1f3ef1d8-2159757.jpg)

    - addAll(Collection<? extends E> c)方法，尾部按顺序添加所有集合元素

      ![img](https://img.mubu.com/document_image/905be62b-b6fa-4faa-a030-95a6088ad4a1-2159757.jpg)

    - addAll(int index, Collection<? extends E> c)方法，指定位置按顺序添加所有集合元素

      ![img](https://img.mubu.com/document_image/6d39633a-88c5-47b8-a1a8-4fff201ab467-2159757.jpg)

  - 元素读取：

    ![img](https://img.mubu.com/document_image/84bcca88-06db-4105-9aa8-50bebbd80ca2-2159757.jpg)

  - 元素删除：关于ArrayList元素的删除，提供了5个方法

    ![img](https://img.mubu.com/document_image/9f2370e8-d321-4149-9ae5-220a4002ea7d-2159757.jpg)

    - remove(int index)方法，删除指定位置的元素

      ![img](https://img.mubu.com/document_image/ad8fca52-e347-48d7-9e89-ee17a5eb79c4-2159757.jpg)

    - remove(Object o)方法，删除数组中第一个为o的元素

      ![img](https://img.mubu.com/document_image/98509b98-b61a-4d1d-8861-e63795cc2b28-2159757.jpg)

    - clear()方法，删除列表中所有的元素

      ![img](https://img.mubu.com/document_image/10c80c5f-c530-4405-a112-a012d3334157-2159757.jpg)

    - removeAll方法，删除传入参数集合中的所有元素

      ![img](https://img.mubu.com/document_image/24969aeb-4963-4864-b990-bc9048ba17e5-2159757.jpg)

    - retainAll方法，删除非传入参数集合中的所有元素

      ![img](https://img.mubu.com/document_image/73a1b70a-89bb-42bf-91d9-049dfe44f160-2159757.jpg)

  - 获取元素索引值

    - indexOf

      ![img](https://img.mubu.com/document_image/a84eb118-1ea1-49b8-91b4-11d3c58d70ba-2159757.jpg)

    - lastIndexOf

      ![img](https://img.mubu.com/document_image/0bffe139-2299-4e3c-ad59-5fe67e86d14a-2159757.jpg)

  - ArrayList转化为数组

    - Object[] toArray()

      ![img](https://img.mubu.com/document_image/14c0abbf-fa00-45df-9fce-fea2eef03b6c-2159757.jpg)

    - <T> T[] toArray(T[] a) 泛型转化

      ![img](https://img.mubu.com/document_image/494afa20-18f5-4c30-bea0-bad4faca20d7-2159757.jpg)