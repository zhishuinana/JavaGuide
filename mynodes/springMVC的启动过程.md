SpringMVC应用启动过程

以下是我们熟知的 SpringMVC 项目中 web.xml 的基础配置，其关键是要配置一个 ContextLoaderListener 和一个 DispatcherServlet。

```xml
<web-app>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring.xml</param-value>
    </context-param>

    <!-- ContextLoaderListener -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- DispatcherServlet -->
    <servlet>
        <description>spring mvc servlet</description>
        <servlet-name>springMvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
          <description>spring mvc</description>
          <param-name>contextConfigLocation</param-name>
          <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springMvc</servlet-name>
        <url-pattern>*.do</url-pattern>
    </servlet-mapping>
</web-app>
```

##### 1.寻找IOC容器实现类

Java Web 容器中相关组件的启动顺序是 ServletContext -> listener -> filter -> servlet ， listener 是优于 servlet 启动的，所以我们先看一看 ContextLoaderListener 的内容。

```java
package org.springframework.web.context;

import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;

public class ContextLoaderListener extends ContextLoader implements ServletContextListener {

	public ContextLoaderListener() {
	}

	public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}

	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}

	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```

ContextLoaderListener继承自**ContextLoader**，实现的是**ServletContextListener**接口。

**继承ContextLoader有什么作用？**
ContextLoaderListener可以指定在Web应用程序启动时载入Ioc容器，正是通过ContextLoader来实现的，可以说是Ioc容器的初始化工作。
**实现ServletContextListener又有什么作用？**
ServletContextListener接口里的函数会结合Web容器的生命周期被调用。因为ServletContextListener是ServletContext的监听者，如果ServletContext发生变化，会触发相应的事件，而监听器一直对事件监听，如果接收到了变化，就会做出预先设计好的相应动作。由于ServletContext变化而触发的监听器的响应具体包括：在服务器启动时，ServletContext被创建的时候，服务器关闭时，ServletContext将被销毁的时候等，相当于web的生命周期创建与效果的过程。

**那么ContextLoaderListener的作用是什么？**

ContextLoaderListener因为实现了**ServletContextListener**接口，可以监听web容器（即ServletContext）是否启动或者销毁，一旦web容器启动，ContextLoaderListener的contextInitialized方法会被调用，该方法的内容是继续调用 initWebApplicationContext 方法，于是我们再跟踪 initWebApplicationContext。(ContextLoaderListener 是 ContextLoader 的子类，所以其实是调用了父类ContextLoader的 initWebApplicationContext 方法。该方法主要是创建IOC容器并初始化)

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {			 
 // application对象中存放了spring context，则抛出异常其中ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT"; 
   if(servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
	throw new IllegalStateException("Cannot initialize context because there is already a 		root application context present - " +"check whether you have multiple 			    	  ContextLoader* definitions in your web.xml!");
 }
 Log logger = LogFactory.getLog(ContextLoader.class);
 servletContext.log("Initializing Spring root WebApplicationContext");
 if (logger.isInfoEnabled()) {
     logger.info("Root WebApplicationContext: initialization started");
 }
 long startTime = System.currentTimeMillis();
 try {
     // 创建得到WebApplicationContext，createWebApplicationContext最后返回值被强制转换为ConfigurableWebApplicationContext类型
     if (this.context == null) {
 	     this.context = createWebApplicationContext(servletContext);
     }
 }
 ...
}
```

此处关注 ContextLoader类的createWebApplicationContext 方法，

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // determineContextClass方法中获取contextClass ，即返回IOC容器实现类
    Class<?> contextClass = determineContextClass(sc);
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" +                             contextClass.getName() +"] is not of type [" +                                           ConfigurableWebApplicationContext.class.getName() + "]");
	}
    // 根据contextClass返回实例，即IOC容器实例，强制转换成ConfigurableWebApplicationContext类型
	return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

从代码可知，方法中的逻辑主要是调用  ContextLoader类的determineContextClass 获取 contextClass ，然后根据 contextClass 创建 IOC 容器实例。所以， contextClass 的值将是关键。继续跟进 ContextLoader类的determineContextClass方法，

```java
protected Class<?> determineContextClass(ServletContext servletContext) {
    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
    if (contextClassName != null) {
        try {
		    return ClassUtils.forName(contextClassName,                                                   ClassUtils.getDefaultClassLoader());
		} catch (ClassNotFoundException ex) {
		    throw new ApplicationContextException("Failed to load custom context class ["                + contextClassName + "]", ex);
		}
	} else {
        // defaultStrategies的值是在本类中的static方法中注入的,即该类加载过程中defaultStrategies            已经被赋值,本类的开始部分有static代码块
	    contextClassName =                                                                       defaultStrategies.getProperty(WebApplicationContext.class.getName());
		try {
            // 返回IOC容器实现类
		    return ClassUtils.forName(contextClassName,                                                  ContextLoader.class.getClassLoader());
		} catch (ClassNotFoundException ex) {
				throw new ApplicationContextException("Failed to load default context                     class [" + contextClassName + "]", ex);
		}
	}
}
```

可以看到， contextClassName 是从 defaultStrategies 中获取的，而关于 defaultStrategies 的赋值需要追溯到 ContextLoader 类中的静态代码块,

```java
private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";
private static final Properties defaultStrategies;

static {
    // DEFAULT_STRATEGIES_PATH的值是ContextLoader.properties
    try {
	    ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH,               ContextLoader.class);
		defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
	} catch (IOException ex) {
	    throw new IllegalStateException("Could not load 'ContextLoader.properties': "                + ex.getMessage());
	}
}
```

defaultStrategies 是从 resource 中获取的参数，而 resource 又是从 DEFAULT_STRATEGIES_PATH 中获取，查看可知 DEFAULT_STRATEGIES_PATH 的值是 ContextLoader.properties ，ContextLoader.class同级目录下面找到ContextLoader.properties文件，其中内容如下

```properties 
org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```

由此可知， SpringMVC 项目中使用到的 IOC 容器类型是 XmlWebApplicationContext。

##### 2.寻找关键入口方法refresh()

我们回到 ContextLoader 的 initWebApplicationContext 方法，前边我们说到调用 createWebApplicationContext 方法创建容器，容器创建后我们关注的下一个方法是 ContextLoader类的configureAndRefreshWebApplicationContext，

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    ...
    try {
	    if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			if (!cwac.isActive()) {
				if (cwac.getParent() == null) {
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
                /** 配置和刷新容器 **/
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
        ...
    }
}
```

configureAndRefreshWebApplicationContext的代码如下

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        } else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }
    wac.setServletContext(sc);
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }
    customizeContext(sc, wac);
    // 调用容器的refresh()方法，此处wac对应的类是XmlWebApplicationContext
    wac.refresh();
}
```

在这里我们要关注的是最后一行 wac.refresh() ，意思是调用容器的 refresh() 方法，此处我们的容器是XmlWebApplicationContext，对应的 refresh() 在其父类 AbstractApplicationContext，

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            ...
        }
        catch (BeansException ex) {
            ...
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

##### 3.加载与注册

在这个 refresh() 方法中，我们首先关注的是 obtainFreshBeanFactory()，

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

obtainFreshBeanFactory() 方法中调用了 refreshBeanFactory() ，而这个方法是在 AbstractApplicationContext 的子类 AbstractRefreshableApplicationContext 实现，

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        //	调用子类XmlWebApplicationContext的loadBeanDefinitions方法
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

refreshBeanFactory方法中调用了 loadBeanDefinitions 方法，路线又回到了 XmlWebApplicationContext 容器，loadBeanDefinitions 方法是在 XmlWebApplicationContext 类中实现的，

```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    // 注意传入的beanDefinitionReader是XmlBeanDefinitionReader
    loadBeanDefinitions(beanDefinitionReader);
}
```

方法最后是调用了重载的 loadBeanDefinitions 方法，传入的参数是 **XmlBeanDefinitionReader** 的对象，我们先看一看重载的 loadBeanDefinitions 方法，

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        for (String configLocation : configLocations) {
            reader.loadBeanDefinitions(configLocation);
        }
    }
}
```

可以看到，其实最后是调用 reader 的 loadBeanDefinitions 方法，此处 reader 的类型是 XmlBeanDefinitionReader ，所以我们查看 XmlBeanDefinitionReader 中的 loadBeanDefinitions 方法，

```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isInfoEnabled()) {
        logger.info("Loading XML bean definitions from " + encodedResource);
    }

    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<EncodedResource>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
                "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // do开头的方法表示是正真执行处理操作的方法
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
                "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}
```

此处我们关注的是 doLoadBeanDefinitions 方法，有个技巧，在Spring中，以do开头的方法都是最终实际执行逻辑处理的。

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
        throws BeanDefinitionStoreException {
    try {
        // 读取加载配置文件
        Document doc = doLoadDocument(inputSource, resource);
        // 注册Bean
        return registerBeanDefinitions(doc, resource);
    } catch (BeanDefinitionStoreException ex) {
        throw ex;
    } catch (SAXParseException ex) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
    } catch (SAXException ex) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                "XML document from " + resource + " is invalid", ex);
    } catch (ParserConfigurationException ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                "Parser configuration exception parsing XML from " + resource, ex);
    } catch (IOException ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                "IOException parsing XML document from " + resource, ex);
    } catch (Throwable ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                "Unexpected exception parsing XML document from " + resource, ex);
    }
}
```

在这里有两个方法值得我们关注，一个是 doLoadDocument ，负责把配置文件读取为 Document 对象，这个方法中包含了「定位」过程的处理逻辑，关于定位过程我们在最后再详细分析；第二个是 registerBeanDefinitions 方法，这个方法包含了「加载」和「注册」逻辑。

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    	/** 获取BeanDefinitionDocumentReader,此处获得的对象实际类型为DefaultBeanDefinitionDocumentReader **/
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    	/** 调用Reader的registerBeanDefinitions方法 */
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

这里关键有两步，第一步是获取 documentReader ，第二步是调用 documentReader 的 registerBeanDefinitions 方法。稍微跟踪一下可知 documentReader 的实际类型是 DefaultBeanDefinitionDocumentReader，所以我们进入到 DefaultBeanDefinitionDocumentReader 的 registerBeanDefinitions 方法，

```java
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    Element root = doc.getDocumentElement();
    doRegisterBeanDefinitions(root);
}
```

按照惯例，干活的是do开头的方法，

```java
protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);

    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                            "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }

    preProcessXml(root);
    /** 解析转换为BeanDefinitions **/
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);

    this.delegate = parent;
}
```

然后是 parseBeanDefinitions 方法

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                /**判断元素是否属于默认的Namespace（当标签为<beans>时判断条件为真） **/
                if (delegate.isDefaultNamespace(ele)) {
                    /** 处理默认的Element **/
                    parseDefaultElement(ele, delegate);
                }
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```

isDefaultNamespaces 方法是判断元素是否属于默认的 Namespace ，通过跟踪可知，这个默认的 Namespace 是指 `<beans>` 标签， 我们知道在spring的配置文件中，对bean的定义是放在 `<beans>` 标签里边的，所以接下来的 parseDefaultElement 方法则是用于解析 bean 定义的。

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    /** 处理<import>标签 **/
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    /** 处理<alias>标签 **/
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    /** 处理<bean>标签 **/
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);
    }
    /** 处理嵌套的<beans>标签 **/
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // recurse
        doRegisterBeanDefinitions(ele);
    }
}
```

到这里我们就一目了然了，这很显然就是针对 `<beans>` 标签中的各种元素进行解析，对于其他标签我们不深究，直接看处理 `<bean>` 标签的 processBeanDefinition 方法

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    /**
	 * 调用BeanDefinitionParserDelegate的parseBeanDefinitionElement方法
	 * 返回一个包含BeanDefinition信息的BeanDefinitionHolder实例
	 * **/
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            /**
			 * 注册BeanDefinition
			 * 传入的参数是刚刚获取到的BeanDefinitionHolder对象，再加上DefaultListableBeanFactory对象
			 * DefaultListableBeanFactory对象的由来需要追溯到AbstractRefreshableApplicationContext的refreshBeanFactory()方法中
			 * **/
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

这里有两个非常关键的方法：

- 一个是 parseBeanDefinitionElement ，这个方法最终会返回一个持有 BeanDefinition 的 BeanDefinitionHolder 实例，我们在上篇开头的结论中已经说了，加载的过程其实就是把bean的定义转换成一个 BeanDefinition 对象，所以 parseBeanDefinitionElement 对应的便是 「加载」 过程；
- 另一个则是 registerBeanDefinition ，这个方法对应的便是「注册」过程。

接下来我们将分两部分分别讲解 parseBeanDefinitionElement 和 registerBeanDefinition 的内容。

###### 加载(parseBeanDefinitionElement )

`BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);` 中的 delegate 是 BeanDefinitionParserDelegate 的实例，我们查看 BeanDefinitionParserDelegate 中的 parseBeanDefinitionElement方法.

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}
```

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
    ...
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    ...
    return null;
}
```

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
        Element ele, String beanName, BeanDefinition containingBean) {

    this.parseState.push(new BeanEntry(beanName));

    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }

    try {
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);
        /** 解析bean定义的属性 **/
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

        parseMetaElements(ele, bd);
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        parseConstructorArgElements(ele, bd);
        parsePropertyElements(ele, bd);
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;
    }
    catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    }
    catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    }
    catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    }
    finally {
        this.parseState.pop();
    }

    return null;
}
```

这里的parse开头的方法都是对bean定义的属性标签进行解析，例如「name」、「singleton」、「lazy-init」等，大家可以自行深入了解每一个parse方法是如何解析各个属性的，此次不再逐一讲解了。至此我们已经获取到了BeanDefinition的信息，下一步就到「注册」了。

###### 注册(registerBeanDefinition )

所谓的注册，其实就是把BeanDefintion存储到IOC容器中，我们进入到 registerBeanDefinition 中看一看是如何实现的。

```java
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    	/** 此处实际是调用DefaultListableBeanFactory的registerBeanDefinition方法 **/
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

这里最终调用的是 registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition()) 其中 registry 是 DefaultListableBeanFactory 的实例。

（为什么是 DefaultListableBeanFactory ？在 AbstractRefreshableApplicationContext 的 refreshBeanFactory() 方法中会创建DefaultListableBeanFactory的实例，并在之后的所有关键方法中都会作为参数传入该实例，保证后续的调用流程中都能获取到该实例）

DefaultListableBeanFactory 的 registerBeanDefinition 方法如下

```
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {
	...
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        ...
        this.beanDefinitionMap.put(beanName, beanDefinition);
    } else {
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
            /**
			 * 把BeanDefinitionc添加到beanDefinitionMap中
			 * Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
			 * 由此可知，IOC的启动过程是先把Bean的定义解析转换为BeanDefiniton，
			 * 最后存储于IOC容器（DefaultListableBeanFactory是一个IOC容器）的一个Map变量中。
			 * */
            this.beanDefinitionMap.put(beanName, beanDefinition);
            /**
			 * 把所有的Bean名存储于beanDefinitionNames列表中
			 */
            this.beanDefinitionNames.add(beanName);
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

`this.beanDefinitionMap.put(beanName, beanDefinition);` 这一行是重点，字面意思已经很明显，就是把 beanName 和 beanDefinition 以 key-value 的形式存储于 beanDefinitionMap 中， beanDefinitionMap 的定义如下。

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
```

跟踪到这一步我们便可以得出结论：「 BeanDefinition 是存储在 DefaultListableBeanFactory 的一个 Map 数据结构中 」

##### 4.定位

![](E:\sourceCode\JavaGuide\mynodes\img\SpringIOC.jpg)

 入口ContextLoader 的 configureAndRefreshWebApplicationContext 方法。

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }

    wac.setServletContext(sc);
    /**
     * CONFIG_LOCATION_PARAM = "contextConfigLocation"
     * 对应的含义是读取web.xml中contextConfigLocation的值
     */
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }

    customizeContext(sc, wac);
    wac.refresh();
}
```

`wac.setConfigLocation(configLocationParam);`。configLocationParam 的值是调用 `sc.getInitParameter(CONFIG_LOCATION_PARAM);` 获取的，CONFIG_LOCATION_PARAM的值是 `"contextConfigLocation"` ，对于 `"contextConfigLocation"` 有印象吗？回顾我们的 web.xml 配置文件中可以看到，我们配置了一个 `<context-param>` 上下文参数，这个参数名便是 `contextConfigLocation` ， 而其值是便是我们自定义的 Spring 配置文件路径。

```java
<web-app>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring.xml</param-value>
    </context-param>

    <!-- ContextLoaderListener -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- DispatcherServlet -->
   ...
</web-app>
```

还可以继续深入查看 setConfigLocation 的处理逻辑，其实现过程是在 AbstractRefreshableConfigApplicationContext 类中，通过代码可知道配置文件支持的分隔符有`,; \t\n`。

接下来我们直接分析 XmlWebApplicationContext 的 loadBeanDefinitions(XmlBeanDefinitionReader) 方法，

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
    /** 获取配置文件位置*/
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        for (String configLocation : configLocations) {
            /** 实际是调用XmlBeanDefinitionReader的loadBeanDefinitions方法 **/
			reader.loadBeanDefinitions(configLocation);
		}
	}
}
```

这个方法我们在前边也是看过的，当时我们只关注 `reader.loadBeanDefinitions(configLocation);` 这一行，这次我们关注第一行 `String[] configLocations = getConfigLocations();` ,从方法的字面意思就很容易理解它的作用——获取配置位置，很明显这就是我们的「定位」过程。
getConfigLocations 的实现方法是在其父类 AbstractRefreshableConfigApplicationContext 中

```java
protected String[] getConfigLocations() {
    return (this.configLocations != null ? this.configLocations : getDefaultConfigLocations());
}
```

这个方法的意思是，当容器中的 configLocations 变量为空时则调用 getDefaultConfigLocations， 当不为空时直接返回容器中的 configLocations 。我们先看一看 XmlWebApplicationContext类中的getDefaultConfigLocations 方法，

```java
@Override
protected String[] getDefaultConfigLocations() {
    if (getNamespace() != null) {
        return new String[] {DEFAULT_CONFIG_LOCATION_PREFIX + getNamespace() + DEFAULT_CONFIG_LOCATION_SUFFIX};
    }
    else {
        return new String[] {DEFAULT_CONFIG_LOCATION};
    }
}
```

DEFAULT_CONFIG_LOCATION 的定义是

```java
public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";
```

`/WEB-INF/applicationContext.xml` 便是我们再熟悉不过的 Spring 默认的配置文件路径与文件名。
而当我们再 web.xml 配置了 contextConfigLocation ，Spring则会读取我们的自定义 Spring 配置文件。至此，关于「定位」的过程已经完整讲解完毕。