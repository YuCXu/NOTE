- #ThreadLocal

- **一、ThreadLocal是什么？有什么用？**

  - 引入话题：在并发条件下，如何正确获得共享数据？举例：假设有多个用户需要获取用户信息，一个线程对应一个用户。在mybatis中，session用于操作数据库，那么设置、获取操作分别是session.set()、session.get()，如何保证每个线程都能正确操作达到想要的结果？

    ![img](https://img.mubu.com/document_image/5d21f2ec-cb62-4a13-bede-ee7dbbadfa0a-2159757.jpg)

  - 有synchronized：

    ![img](https://img.mubu.com/document_image/2e4bfa91-3f8b-4921-a096-4b2f27d46ba5-2159757.jpg)

  - 没有synchronized：

    ![img](https://img.mubu.com/document_image/da46ac2c-9b4c-4868-8a6d-ed3636551a07-2159757.jpg)

  - person是共享数据，用同步互斥锁synchronized，当一个线程访问共享数据的时候，其他线程堵塞。

    ![img](https://img.mubu.com/document_image/7efafe8b-a17e-48ca-b3d0-fd929cb4f960-2159757.jpg)

  - 运行结果：

    ![img](https://img.mubu.com/document_image/03fccf9e-be46-4272-96b8-9cd26d7c33ba-2159757.jpg)

  - 回顾java内存模型：

    ![img](https://img.mubu.com/document_image/0a23e733-1d31-4556-a468-11d82508ce67-2159757.jpg)

  - 在虚拟机中，堆内存用于存储共享数据（实例对象），堆内存也就是这里说的主内存。

  - 每一个线程将会在堆内存中开辟一块空间叫做线程的工作内存，附带一块缓存区用于存储共享数据副本。那么，共享数据在堆内存当中，线程通信就是通过主内存为中介，线程在本地内存读并且操作完共享变量后，把值写入主内存。

    - ①ThreadLocal称为线程局部变量，就是线程工作内存的一小块内存，用于存储数据。
    - ②ThreadLocal.set()、ThreadLocal.get()方法，就相当于把数据存储于线程本地，取也是在本地内存读取。就不会像synchronized需要频繁的修改主内存的数据，再把数据复制到工作内存，也大大提供访问效率。

  - ThreadLocal的作用：

    - 作用一：因为线程间的数据交互是通过工作内存与主内存的频繁读写完成通信，然而存储于线程本地内存，提高访问效率，避免线程阻塞造成cpu吞吐量下降。
    - 作用二：在多线程中，每一个线程都需要维护session，轻易完成对线程独享资源的操作。

- ThreadLocal是使用空间换时间，synchronized是使用时间换空间，比如在hibernate中session就存在与ThreadLocal中，避免synchronized的使用。

- **二、ThreadLocal实现原理：**

  ![img](https://img.mubu.com/document_image/9f971111-9027-44fc-9a33-cc863d980d2f-2159757.jpg)

  - 每个Thread维护一个ThreadLocalMap映射表，这个映射表的key是ThreadLocal实例本身，value是真正需要存储的Object。
  - 也就是说ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获得value。值得注意的是图中的虚线，表示 ThreadLocalMap 是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。

- 三、ThreadLocal为什么会内存泄漏：

  - ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统GC的是时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value永远无法回收，造成内存泄漏。
  - 其实，ThreadLocalMap的设计中已经考虑到这种情况，也加上了一些防护措施：在ThreadLocal的get()，set()，remove()的时候都会清除线程ThreadLocalMap里所有key为null的value。
  - 但是这些被动的预防措施并不能保证不会内存泄漏：
    - 使用static的ThreadLocal，延长了ThreadLocal的生命周期，可能导致的内存泄漏（参考ThreadLocal 内存泄露的实例分析）。
    - 分配使用了ThreadLocal又不再调用get()，set()，remove()方法，那么就会导致内存泄漏。