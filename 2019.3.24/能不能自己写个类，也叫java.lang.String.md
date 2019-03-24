##### 能不能自己写个类，也叫java.lang.String？

##### 可以，但是即使你写了这个类，也没有用。

这个问题涉及到类加载器的委托机制，在类加载器的结构图中，BootStrapClassLoader是顶层父类，ExtensionClassLoader是BootStrapClassLoader类的子类，ExtensionClassLoader又是ApplicationClassLoader的父类。以java.lang.String为例，当我使用这个类时，java虚拟机会将java.lang.String类的字节码加载到内存中。

##### 为什么只加载系统通过的java.lang.String类而不加载用户自定义的java.lang.String类呢？

因为加载某个类时，优先使用父类加载器加载需要使用的类。如果我们自定义了java.lang.String这个类，加载该**自定义的String类，所使用的类加载器是ApplicationClassLoader**，根据优先使用父类加载器原理，ApplicationClassLoader加载器的父类为ExtensionClassLoader，所以这时加载String使用的类加载器是ExtensionClassLoader，但是类加载器ExtensionClassLoader在jre/lib/ext目录下**没有找到**String.class类。然后使用ExtensionClassLoader父类加载器BootStrapClassLoader，父类加载器BootStrapClassLoader在jre/lib目录的rt.jar**找到了**String.class，将其加载到内存中，这就是类加载器的委托机制。