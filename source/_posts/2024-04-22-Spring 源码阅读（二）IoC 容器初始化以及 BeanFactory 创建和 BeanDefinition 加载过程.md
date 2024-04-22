---
title: Spring 源码阅读（二）IoC 容器初始化以及 BeanFactory 创建和 BeanDefinition 加载过程
date: 2024-04-22 22:56:11 
category: Spring
id: spring-source-02
---

> [相关代码提交记录：https://github.com/linweiwang/spring-framework-5.3.33](https://github.com/linweiwang/spring-framework-5.3.33/commit/2187bc6c7b3a0ff2438a0c0fd3bbfef8b17f331e)


## IoC 容器三种启动方式

**XML**

JavaSE：
```java
ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml")
ApplicationContext context = new FileSystemXmlApplicationContext("C:/beans.xml")
```


JavaWeb
```
通过 web.xml 配置 ContextLoaderListener，指定 Spring 配置文件。
```


**XML+注解**

因为有 XML ，所以和纯 XML 启动方式一样


**注解**

JavaSE
```java
ApplicationContext context = new AnnotationConfigApplicationContext(SpringConfig.class)
```

JavaWeb
```
通过 web.xml 配置 ContextLoaderListener，指定 Spring 配置文件。
```


BeanFactory 是 Spring 框架中 IoC 容器的顶层接⼝,它只是⽤来定义⼀些基础功能,定义⼀些基础规范,⽽ ApplicationContext 是它的⼀个⼦接⼝，所以 ApplicationContext 是具备 BeanFactory 提供的全部功能力的。
通常，我们称 BeanFactory 为 SpringIOC 的基础容器，ApplicationContext 是容器的⾼级接⼝，⽐
BeanFactory 要拥有更多的功能，⽐如说国际化⽀持和资源访问（XML、Java 配置类）等等。


![](attachments/SpringSource_02_BeanFactoryHierarchy.png)

下面以纯 XML 依赖原有 spring-research 来跟踪源码。

## IoC 容器初始化主体流程


分析 `new ClassPathXmlApplicationContext("spring-config.xml");`

ClassPathXmlApplicationContext.java

```java
    // 创建 ClassPathXmlApplicationContext，加载 XML 中的定义信息，并且自动 refresh 容器（Context）
	public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		// 调用重载方法
		this(new String[] {configLocation}, true, null);
	}

	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		// 初始化父类
		super(parent);
		// 设置配置文件
		setConfigLocations(configLocations);
		// refresh context
		if (refresh) {
			refresh();
		}
	}
```

进入 refresh() 方法，在父类 AbstractApplicationContext 中
```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		// 对象锁
		synchronized (this.startupShutdownMonitor) {
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			// 刷新前的预处理
			// Prepare this context for refreshing.
			prepareRefresh();

			// 获取 BeanFactory: 默认实现是 DefaultListableBeanFactory
			// 加载 BeanDefinition 并注册到 BeanDefinitionRegistry
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// BeanFactory 预准备工作
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// BeanFactory 准备工作完成后的后置处理，留给子类实现
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// 实例化实现了 BeanFactoryPostProcessor 接口的 Bean，并调用该接口方法
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
				// 注册 BeanPostProcessor （Bean 的后置处理器），在创建 Bean 的前后执行
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();

				// 初始化 MessageSource 组件：国际化、消息绑定、消息解析等
				// Initialize message source for this context.
				initMessageSource();

				// 初始化事件派发器
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// 初始化其他特殊的 Bean，在容器刷新的时候子类自定义实现：如创建 Tomcat、Jetty 等 Web 服务器
				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// 注册应用监听器（即实现了 ApplicationListener 接口的 Bean）
				// Check for listener beans and register them.
				registerListeners();

				// 初始化创建非懒加载的单例 Bean、填充属性、调用初始化方法（ afterPropertiesSet，init-method 等）、调用 BeanPostProcessor 后置处理器
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// 完成 context 刷新，调用 LifecycleProcessor 的 onRefresh 方法并发布 ContextRefreshedEvent
				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
				contextRefresh.end();
			}
		}
	}
```

整体流程如下：

**1 刷新前的预处理： prepareRefresh();**
 主要是一些准备工作设置其启动日期和活动标志以及执行一些属性的初始化。
 
 **2 初始化 BeanFactory： obtainFreshBeanFactory()；**
- 如果有旧的 BeanFactory 就删除并创建新的 BeanFactory
- 解析所有的 Spring 配置文件，将配置文件中定义的 bean 封装成 BeanDefinition，加载到BeanFactory 中（这里只注册，不会进行 Bean 的实例化）

**3 BeanFactory 预准备工作：prepareBeanFactory(beanFactory);**
配置 BeanFactory 的标准上下文特征，例如上下文的 ClassLoader、后置处理器等。

**4 BeanFactory 准备工作完成后的后置处理，留给子类实现：postProcessBeanFactory(beanFactory);**
空方法，如果子类需要，自己去实现 

**5 调用 Bean 工厂后置处理器：invokeBeanFactoryPostProcessors(beanFactory);**

实例化和调用所有BeanFactoryPostProcessor，完成类的扫描、解析和注册。
BeanFactoryPostProcessor 接口是 Spring 初始化 BeanFactory 时对外暴露的扩展点，Spring IoC 容器允许 BeanFactoryPostProcessor 在容器实例化任何 bean 之前读取 bean 的定义，并可以修改它。

**6 注册 BeanPostProcesso：registerBeanPostProcessors(beanFactory);**
所有实现了 BeanPostProcessor 接口的类注册到 BeanFactory 中。

**7 初始化 MessageSource 组件：initMessageSource();**
初始化MessageSource组件（做国际化功能；消息绑定，消息解析）

**8 初始化事件派发器：initApplicationEventMulticaster();**
初始化应用的事件派发/广播器 ApplicationEventMulticaster。

**9 初始化其他特殊的 Bean：onRefresh();**
空方法，模板设计模式;子类重写该方法并在容器刷新的时候自定义逻辑。
例：SpringBoot 在 onRefresh() 完成内置 Tomcat 的创建及启动

**10 注册应用监听器：registerListeners();**
向事件分发器注册硬编码设置的 ApplicationListener，向事件分发器注册一个 IoC 中的事件监听器（并不实例化）

**11 初始化创建非懒加载的单例 Bean：finishBeanFactoryInitialization(beanFactory);**
初始化创建非懒加载的单例 Bean、填充属性、调用初始化方法（ afterPropertiesSet，init-method 等）、调用 BeanPostProcessor 后置处理器，是整个 Spring IoC 核心中的核心。

**12 完成 context 刷新：finishRefresh();**
完成 context 刷新，调用 LifecycleProcessor 的 onRefresh 方法并发布 ContextRefreshedEvent

## 获取 BeanFactory 子流程

`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`

AbstractApplicationContext.java
```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		// 对 BeanFactory 进行刷新操作，默认实现 AbstractRefreshableApplicationContext#refreshBeanFactory
		refreshBeanFactory();
		// 返回上一步 beanFactory，默认实现 AbstractRefreshableApplicationContext#getBeanFactory
		return getBeanFactory();
	}
```

AbstractRefreshableApplicationContext
```java
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		// 判断是否已有 BeanFactory
		if (hasBeanFactory()) {
			// 销毁 Bean
			destroyBeans();
			// 关闭 BeanFactory
			closeBeanFactory();
		}
		try {
			// 实例化 DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			// 设置序列化 ID
			beanFactory.setSerializationId(getId());
			// 自定义 BeanFactory 的一些属性：allowBeanDefinitionOverriding、allowCircularReferences
			//   allowBeanDefinitionOverriding 是否允许覆盖
			//   allowCircularReferences  是否允许循环依赖
			customizeBeanFactory(beanFactory);
			// 加载应用中的 BeanFactory
			loadBeanDefinitions(beanFactory);
			// 赋值给当前 beanFactory 属性
			this.beanFactory = beanFactory;
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}

	@Override
	public final ConfigurableListableBeanFactory getBeanFactory() {
		DefaultListableBeanFactory beanFactory = this.beanFactory;
		if (beanFactory == null) {
			throw new IllegalStateException("BeanFactory not initialized or already closed - " +
					"call 'refresh' before accessing beans via the ApplicationContext");
		}
		return beanFactory;
	}
```

![](attachments/SpringSource_02_obtainBeanFactory.png)

## BeanDefinition 加载解析及注册子流程



继续分析 `AbstractRefreshableApplicationContext#refreshBeanFactory 中的 loadBeanDefinitions` 的实现方法在 `AbstractXmlApplicationContext#loadBeanDefinitions` 中（若是注解在 AnnotationConfigWebApplicationContext）

AbstractXmlApplicationContext
```java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 给指定的 BeanFactory 创建一个 XmlBeanDefinitionReader 进行读取和解析 XML
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// 给 XmlBeanDefinitionReader 设置上下文信息
		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// 提供给子类上西安的模板方法：自定义初始化策略
		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		// 真正的去加载 BeanDefinitions
		loadBeanDefinitions(beanDefinitionReader);
	}

	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		// 从 Resource 资源对象加载 BeanDefinitions
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		// 从 XML 配置文件加载 BeanDefinition 对象
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
```
`reader.loadBeanDefinitions(configLocations);` 中调用了 `AbstractBeanDefinitionReader#loadBeanDefinitions`

```java
	@Override
	public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int count = 0;
		// 如果有多个配置文件，循环读取加载，并统计数量
		for (String location : locations) {
			count += loadBeanDefinitions(location);
		}
		return count;
	}

	@Override
	public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(location, null);
	}

	public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		// 获取上下文的 ResourceLoader （资源加载器）
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		// 判断资源加载器是否为 ResourcePatternResolver 类型
		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				// 统一加载转换为 Resource 对象
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				// 通过 Resource 对象加载 BeanDefinitions
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
    		// 否则以绝对路径转换为 Resource 对象
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
	}

	@Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int count = 0;
		for (Resource resource : resources) {
			count += loadBeanDefinitions(resource);
		}
		return count;
	}

	@Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int count = 0;
		for (Resource resource : resources) {
			// 加载
			count += loadBeanDefinitions(resource);
		}
		return count;
	}
```

`loadBeanDefinitions(resource);` 调用了 `XmlBeanDefinitionReader#loadBeanDefinitions` 方法

```java
	@Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}

	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}

		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
			// 把 XML 文件流封装为 InputSource 对象
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			// 执行加载逻辑
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
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

	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
			// 读取 XML 信息，将 XML 中信息保存到 Document 对象中
			Document doc = doLoadDocument(inputSource, resource);
			// 再解析 Document 对象，封装为 BeanDefinition 对象进行注册
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}

	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 获取已有 BeanDefinition 的数量
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 注册 BeanDefinition
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 计算出新注册的 BeanDefinition 数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

`documentReader.registerBeanDefinitions(doc, createReaderContext(resource));` 会进入 `DefaultBeanDefinitionDocumentReader#registerBeanDefinitions`

```java
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}

	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		// 真正解析 XML
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}

	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						// 解析默认标签元素："import", "alias", "bean"
						parseDefaultElement(ele, delegate);
					}
					else {
						// 解析自定义标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			// 解析自定义标签
			delegate.parseCustomElement(root);
		}
	}

	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		// import 标签
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		// alias 标签
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		// bean 标签：着重分析
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		// 嵌套 bean 标签
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		// 解析 bean 标签为 BeanDefinitionHolder 里面持有 BeanDefinition
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			// 若有必要装饰 bdHolder （bean 标签内有自定义标签的情况下）
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 完成 BeanDefinition 的注册
				// Register the final decorated instance.
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

`BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());` 调用了 `BeanDefinitionReaderUtils#registerBeanDefinition`
```java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

        // 注册
		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
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

`registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());` 调用了 `DefaultListableBeanFactory#registerBeanDefinition` 

```java
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (logger.isInfoEnabled()) {
					logger.info("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			// 注册逻辑
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
		else if (isConfigurationFrozen()) {
			clearByTypeCache();
		}
	}
```

整体调用链如下
```
AbstractRefreshableApplicationContext#refreshBeanFactory
    AbstractXmlApplicationContext#loadBeanDefinitions
        AbstractBeanDefinitionReader#loadBeanDefinitions // 加载 BeanDefinition
            XmlBeanDefinitionReader#loadBeanDefinitions
                XmlBeanDefinitionReader#doLoadBeanDefinitions // 读取 XML 为 Document
                    XmlBeanDefinitionReader#registerBeanDefinitions // 真正开始注册
                        DefaultBeanDefinitionDocumentReader#registerBeanDefinitions
                            DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions
                                DefaultBeanDefinitionDocumentReader#parseBeanDefinitions
                                    DefaultBeanDefinitionDocumentReader#parseDefaultElement
                                        DefaultBeanDefinitionDocumentReader#processBeanDefinition
                                            BeanDefinitionReaderUtils#registerBeanDefinition
                                                DefaultListableBeanFactory#registerBeanDefinition
```

![](attachments/SpringSource_02_loadBeanDefinitions.png)

