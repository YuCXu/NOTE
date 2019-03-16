# Mybatis

##### 工作原理原型图：

![1](C:\Users\YuChen_Xu\Desktop\工具手册\每日知识点\NOTE\images\1.PNG)

工作原理解析：

mybatis应用程序通过SqlSessionFactoryBuilder从mybatis-config.xml配置文件（也可以用java文件配置的方式，需要添加@Configuration）中构建出SqlSessionFactory（SqlSessionFactory是线程安全的）；然后，SqlSessionFactory的实例直接开启一个SqlSession，再通过SqlSession实例获得Mapper对象并运行Mapper映射的Sql语句，完成对数据库的CRUD和事务提交，之后关闭SqlSession。

流程图解：

![2](C:\Users\YuChen_Xu\Desktop\工具手册\每日知识点\NOTE\images\2.PNG)

流程：

①加载配置文件。需要加载的配置文件包括全局配置文件(SqlMapConfig.xml)和SQL(Mapper.xml)映射文件，其中全局配置文件配置了MyBatis的运行环境信息(数据源、事务等)，SQL映射文件中配置了与SQL执行相关的信息。

②创建**会话工厂**。MyBatis通过读取配置文件的信息来构造出会话工厂(SqlSessionFactory)。

③创建**会话**。拥有了会话工厂，MyBatis就可以通过它来创建会话对象(SqlSession)。会话对象是一个**接口**，该接口中包含了对数据库操作的增删改查方法。

④创建**执行器**。因为会话对象本身不能直接操作数据库，所以它使用了一个叫做数据库执行器(Executor)的接口来帮它执行操作。

⑤封装SQL对象。在这一步，执行器将待处理的SQL信息封装到一个对象中(**MappedStatement**)，该对象包括SQL语句、输入参数映射信息(Java简单类型、HashMap或POJO)和输出结果映射信息(Java简单类型、HashMap或POJO)。

⑥操作数据库。拥有了执行器和SQL信息封装对象就使用它们访问数据库了，最后再返回操作结果，结束流程。

 

 

 

 

 