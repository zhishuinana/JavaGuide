##### Spring的启动流程

定位->加载->注册->实例化

- 定位：获取配置文件路径，通常我们spring的配置文件是 application.xml 或自定义的 spring-xxx.xml 
- 加载：把配置文件读取成BeanDefinition
- 注册：存储BeanDefinition
- 实例化：根据 BeanDefinition 创建实例

IOC容器的关键入口方法是 refresh()。





