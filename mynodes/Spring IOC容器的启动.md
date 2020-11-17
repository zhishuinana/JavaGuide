Spring IOC容器的启动

启动Spring容器,本质上是创建并初始化一个具体的容器类的过程,以常见的容器类ClassPathXmlApplicationContext为例,启动一个Spring容器可以用以下代码表示：

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
```

![img](https://img-blog.csdn.net/20180702013348176?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MTkwMDIz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



