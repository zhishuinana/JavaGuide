# Spring IOC容器的启动

@AutoConfigurationPackage的作用

1. @AutoConfigurationPackage注解的功能是由@Import注解实现的。

2. @Import注解是spring框架的底层注解，它的作用就是给容器中导入某个组件类，例如@Import(AutoConfigurationPackages.Registrar.class)，它就是将Registrar这个组件类注册到springioc容器中；

3. 可查看Registrar类中registerBeanDefinitions方法，这个方法就是导入组件类的具体实现 ，从源码可以看出，在Registrar类中有一个registerBeanDefinitions()方法，使用Debug模式启动项目， 可以看到选中的部分就是com.lagou。也就是说，@AutoConfigurationPackage注解的主要作用就是将主程序类所在包及所有子包下的组件到扫描到spring容器中。

   ```java
   
   ```

   