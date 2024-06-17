
---
title: Spring 源码阅读（三）Bean 的生命周期
date: 2024-06-17 22:46:11 
category: Spring
id: spring-source-03
---

由于 Spring 源码非常多，博客中贴源码会占用大量篇幅，阅读困难。详细分析部分会以 commit 提交形式关联源码提交，画图例来说明源码整体逻辑。
## Bean 生命周期主体逻辑

相关代码：[Bean的基本创建流程、lazyInit、循环依赖](https://github.com/linweiwang/spring-framework-5.3.33/commit/bdcd486b705d96ea10b6bf8e4a0941a33795f2e3)
### Bean 对象创建基本流程

通过最开始的关键时机点分析，我们知道Bean创建⼦流程⼊⼝在 `AbstractApplicationContext#refresh()` ⽅法的 `finishBeanFactoryInitialization(beanFactory)` 处执行的。

整体调用顺序如下：
```
AbstractApplicationContext#refresh
    AbstractApplicationContext#finishBeanFactoryInitialization
        DefaultListableBeanFactory#preInstantiateSingletons
            AbstractBeanFactory#getBean
                AbstractBeanFactory#doGetBean
                    DefaultSingletonBeanRegistry#getSingleton
                    AbstractAutowireCapableBeanFactory#createBean
                        AbstractAutowireCapableBeanFactory#doCreateBean
```


我们从  `AbstractApplicationContext#finishBeanFactoryInitialization` 开始分析

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no BeanFactoryPostProcessor
		// (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// 实例化所有立即加载的单例 Bean
		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

`DefaultListableBeanFactory#preInstantiateSingletons`

```java
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// 存放 BeanNames
		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// 触发所有非延迟加载单例 Bean 的初始化，主要步骤是 getBean
		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			// 合并父 BeanDefinition 对象
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				// 工厂 Bean：&BeanName
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else { // 实例化 Bean
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				StartupStep smartInitialize = getApplicationStartup().start("spring.beans.smart-initialize")
						.tag("beanName", beanName);
				SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
				smartInitialize.end();
			}
		}
	}
```

`AbstractBeanFactory#getBean` 与 `AbstractBeanFactory#doGetBean`
```java
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}

	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		// 解析 BeanName，如果以 & 开头则去掉 & ，如果是别名则获取到真正的名字
		String beanName = transformedBeanName(name);
		Object beanInstance;

		// 从缓存获取 Bean（注意点：三级缓存）
		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		// 如果存在即返回
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 针对 FactoryBean 做处理
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// prototype 类型的 Bean 不支持循环依赖
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查父工厂中是否已存在该对象
			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			// 标记
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
					.tag("beanName", name);
			try {
				if (requiredType != null) {
					beanCreation.tag("beanType", requiredType::toString);
				}
				// 合并父子 Bean 属性
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 处理 dependsOn 配置
				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 创建 Bean 实例
				// Create bean instance.
				if (mbd.isSingleton()) {
					// 单例
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// 创建 Bean
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// 原型
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new ScopeNotActiveException(beanName, scopeName, ex);
					}
				}
			}
			catch (BeansException ex) {
				beanCreation.tag("exception", ex.getClass().toString());
				beanCreation.tag("message", String.valueOf(ex.getMessage()));
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
			finally {
				beanCreation.end();
			}
		}

		return adaptBeanInstance(name, beanInstance, requiredType);
	}
```


`DefaultSingletonBeanRegistry#getSingleton`

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			// 丛单例池中获取 Bean
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				// 是否正在销毁，异常
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				// 验证完真正开始创建对象，先标识该 Bean 正在被创建
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 传进来的 Lambda 表达式： singletonFactory ，并调用 getObject
					// （即上一步骤中的 () -> { try { return createBean(beanName, mbd, args); } catch (BeansException ex) { ... } }）
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

`AbstractAutowireCapableBeanFactory#createBean` 和 `AbstractAutowireCapableBeanFactory#doCreateBean`

```java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		// 拿到 mbd
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// doCreateBean 方法进行创建 Bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}

	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			// 创建 Bean 实例，仅调用构造方法，尚未设置属性
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// 初始化 Bean 实例：属性填充
		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			// 属性填充
			populateBean(beanName, mbd, instanceWrapper);
			// 调用初始化方法，引用 BeanPostProcessor 后置处理器
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

### lazy-init 加载机制的基本流程

在配置文件中设置 lazy-init
```xml
<bean id="person" class="io.github.linweiwang.bean.Person" lazy-init="true">
</bean>
```

从 `DefaultListableBeanFactory#preInstantiateSingletons` 的 `if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {` 可知 lazy-init 的 Bean 在容器启动时是不会进行实例化的，通常是第⼀次进⾏ `context.getBean()` 时进⾏触发

整体调用顺序如下：
```
AbstractApplicationContext#getBean
    DefaultListableBeanFactory#getBean(Class<T> requiredType)
        DefaultListableBeanFactory#getBean
            DefaultListableBeanFactory#resolveBean
                DefaultListableBeanFactory#resolveNamedBean
                    DefaultListableBeanFactory#resolveNamedBean
                        AbstractBeanFactory#getBean
                            AbstractBeanFactory#doGetBean
            
```

`DefaultListableBeanFactory#resolveNamedBean`

```java
	@Nullable
	private <T> NamedBeanHolder<T> resolveNamedBean(
			String beanName, ResolvableType requiredType, @Nullable Object[] args) throws BeansException {
        // AbstractBeanFactory#getBean 里面会调用 doGetBean
		Object bean = getBean(beanName, null, args);
		if (bean instanceof NullBean) {
			return null;
		}
		return new NamedBeanHolder<T>(beanName, adaptBeanInstance(beanName, bean, requiredType.toClass()));
	}
```


### 循环依赖基本流程

循环依赖其实就是循环引⽤，也就是两个或者两个以上的 Bean 互相持有对⽅，最终形成闭环。⽐如A 依赖于B，B 依赖于 C，C ⼜依赖于 A。

单例 Bean 构造器参数循环依赖（⽆法解决）只能拋出 BeanCurrentlyInCreationException 异常。

Prototype 原型 Bean循环依赖（⽆法解决）对于原型 Bean 的初始化过程中不论是通过构造器参数循环依赖还是通过 setXxx ⽅法产⽣循环依赖，Spring 都会直接报错处理。

单例 Bean 的Field 属性的循环依赖（可以解决），Spring 采⽤的是提前暴露对象的⽅法（也即三级缓存）。

示例
```xml
	<bean id="a" class="io.github.linweiwang.bean.A">
		<property name="b" ref="b"/>
	</bean>
	<bean id="b" class="io.github.linweiwang.bean.B">
		<property name="a" ref="a"/>
	</bean>
```

从 `AbstractAutowiredCapableBeanFactory#doGetBean()` 入手
```java
		// 处理循环依赖：单例、允许循环依赖
		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 加入三级缓存
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
		// 初始化 Bean 实例：属性填充
		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			// 属性填充
			populateBean(beanName, mbd, instanceWrapper);
			// 调用初始化方法，引用 BeanPostProcessor 后置处理器
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
```

`AbstractAutowiredCapableBeanFactory#populateBean`
```java
		if (pvs != null) {
			// 处理属性
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
```

`AbstractAutowiredCapableBeanFactory#applyPropertyValues`

```java
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				if (originalValue == AutowiredPropertyMarker.INSTANCE) {
					Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
					if (writeMethod == null) {
						throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
					}
					originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
				}
				// 真正去处理属性值
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
```

`BeanDefinitionValueResolver#resolveValueIfNecessary`

```java
		// 检查属性是否需要运行时引用另外一个 Bean
		// We must check each value to see whether it requires a runtime reference
		// to another bean to be resolved.
		if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
			return resolveReference(argName, ref);
		}
```

`BeanDefinitionValueResolver#resolveReference`

```java
					// 获取 Bean
					resolvedName = String.valueOf(doEvaluate(ref.getBeanName()));
					bean = this.beanFactory.getBean(resolvedName);
```

继续调用 `AbstarctBeanFactory#getBean` 这时候要创建引用的 Bean 即 B。

整体调用流程为：
```
AbstractAutowiredCapableBeanFactory#doGetBean()
    DefaultSingletonBeanRegistry#addSingletonFactory // 加入三级缓存
    AbstractAutowiredCapableBeanFactory#populateBean // 属性填充
        AbstractAutowiredCapableBeanFactory#applyPropertyValues
            BeanDefinitionValueResolver#resolveValueIfNecessary
                BeanDefinitionValueResolver#resolveReference
                    AbstarctBeanFactory#getBean
    
```

下面进入循环：即创建 B  的过程中，需要 A：

创建 B 的时候，发现需要依赖 A，此时 B 依然需经历上述步骤，为了获取 A 到了 `this.beanFactory.getBean(resolvedName)`。 

注意此时处于 B 的初始化过程，B 需要获取 A 。在这个过程中：由于此时 A 已经在三级缓存中了，在获取 A 的 doGetBean 过程中 `Object sharedInstance = getSingleton(beanName);` 会从缓存中获取 A。获取到之后返回给 B。

`DefaultSingletonBeanRegistry#getSingleton`

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		// 从一级缓存 singletonObjects 获取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			// 从二级缓存 earlySingletonObjects 获取
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							// 从三级缓存 singletonFactories 获取
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								singletonObject = singletonFactory.getObject();
								// 从三级缓存 singletonFactories 转移至 二级缓存 earlySingletonObjects
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```


此时 B 已经获取到所有属性后已经完全装配了，在 `DefaultSingletonBeanRegistry#getSingleton` 中调用 `addSingleton(beanName, singletonObject);`

`DefaultSingletonBeanRegistry#addSingleton`
```java
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```

此时 B 已经创建完成。 A 后续也获取到 B 完成创建。

从上述过程中可以分析构造方法的注入是无法循环依赖的，因为无法提前创建出 A 或者 B 放入缓存中。（new 一个对象即便不设置任何属性值也需要调用其构造方法）。

## Bean 生命周期详细分析

所有增强器的创建和调用可以通过在每个方法打断点，通过分析堆栈信息来分析。

### BeanFactory 后置增强

BeanFactory 的后置增强位于 `AbstractApplicationContext#invokeBeanFactoryPostProcessors(beanFactory)`

此部分对应 BeanFactory 和 Bean 的后置增强实现。首先分析下三种后置增强处理器的核心接口
- BeanFactoryPostProcessor：BeanFactory 的后置增强处理器
    - BeanDefinitionRegistryPostProcessor 
- BeanPostProcessor：Bean 的后置增强处理器，可以改变 Bean
    - InstantiationAwareBeanPostProcessor
        - SmartInstantiationAwareBeanPostProcessor
    - MergedBeanDefinitionPostProcessor
    - DestructionAwareBeanPostProcessor
- InitializingBean：Bean 初始化进行后续设置

自定义增强器实现上述结构，并 override 上述接口中的方法。 以某一个 Bean 的创建为例子。
相关代码：[BeanFactory 和 Bean 的后置增强处理](https://github.com/linweiwang/spring-framework-5.3.33/commit/d8afaacff00cd96cc2a2b10065579aa1e0416f67)

![](attachments/SpringSource_03_BeanFactoryPostProcessor_BeanPostProcessor.png)

Spring 底层默认会有一个 BeanDefinitionRegistryPostProcessor 类即：`org.springframework.context.annotation.internalConfigurationAnnotationProcessor` 类型为 `ConfigurationClassPostProcessor`  并且实现了 `PriorityOrdered` 接口。核心方法 `ConfigurationClassPostProcessor#doProcessConfigurationClass`，用于解析 `@Configuration` 配置类，参考代码 [ConfigurationClassPostProcessor 解析配置类](https://github.com/linweiwang/spring-framework-5.3.33/commit/64b6977c332f65086f4da26ee91108b991d5d939)

![](attachments/SpringSource_03_internalConfigurationAnnotationProcessor.png)

### 注册 Bean 后置增强器

Bean 的后置增强的创建位于 `AbstractApplicationContext#registerBeanPostProcessors(beanFactory)` 此处仅作创建，不会调用增强器的方法，参考代码 [Bean 后置增强器的创建](https://github.com/linweiwang/spring-framework-5.3.33/commit/6a28f9010284fe032256d0268fd4a6626a6663dd)

![](attachments/SpringSource_03_registerBeanPostProcessor.png)

### 调用 Bean 的 `SmartInstantiationAwareBeanPostProcessors#predictBeanType` 后置增强器的方法 

Bean 的后置增强的调用位于 `AbstractApplicationContext#registerListeners()` 。相关源码：[SmartInstantiationAwareBeanPostProcessors#predictBeanType 的执行](https://github.com/linweiwang/spring-framework-5.3.33/commit/d71ee7900353b72d74251b80a0abe0cf39aa7bec)

![](attachments/SpringSource_03_predictBeanType.png)

### 初始化创建非懒加载的 Bean

`AbstractApplicationContext#finishBeanFactoryInitialization(beanFactory)` 相关源码： [初始化创建非懒加载的 Bean](https://github.com/linweiwang/spring-framework-5.3.33/commit/d71ee7900353b72d74251b80a0abe0cf39aa7bec)

![](attachments/SpringSource_03_finishiBeanFactoryInitialization.png)

###  AbstractAutowiredCapableBeanFactory#createBean

这一步真正的去创建 Bean，核心调用逻辑 `doCreateBean `，相关源码： [doCreateBean 核心逻辑](https://github.com/linweiwang/spring-framework-5.3.33/commit/4c1e7584d4f8b34a7511cd93979bfe65638a29f9)

createBeanInstance

![](attachments/SpringSource_03_doCreateBean01.png)


三次缓存的处理（后续详细分析）和 createBeanInstance

![](attachments/SpringSource_03_doCreateBean02.png)

initializeBean

![](attachments/SpringSource_03_doCreateBean03.png)
