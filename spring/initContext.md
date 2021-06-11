### 容器启动，初始化上下文
每一个spring项目在配置时，在```web.xml```总少不了这一步：  
```
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```
我们查看```ContextLoaderListener```的实现可知它实现了```ServletContextListener```的两个接口即```contextInitialized```上下文的初始化和```contextDestroyed```上下文的销毁：  
```
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }

    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
    }

    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}

```
我们对上下文的销毁不过多关注，此处只关注上下文的初始化，即```contextInitialized()```，也就是```initWebApplicationContext()```:  
```
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        //启动检查，检查org.springframework.web.context.WebApplicationContext.ROOT是否存在，如果存在则是重复启动，抛异常
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		servletContext.log("Initializing Spring root WebApplicationContext");
		Log logger = LogFactory.getLog(ContextLoader.class);
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// 如果不做特殊设置，此处一般情况下会生成XmlWebApplicationContext.class
            // 或者也可以指定为AnnotationConfigWebApplicationContext.class，
            // 也就是通过注解，但是它们都是ConfigurableWebApplicationContext.class的子类
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                //此处的isActive()获取的是AtomicBoolean.get()，因为此处还未赋值，所以此处是false，也就是还未refresh
				if (!cwac.isActive()) {
					// 初始化此处应为null
					if (cwac.getParent() == null) {
						// 默认实现返回null
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
            //启动完成，防止org.springframework.web.context.WebApplicationContext.ROOT标识
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException | Error ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
	}
```
以上方法只是对上下文创建的检测和初始化，具体资源文件的加载，配置文件的解析，bean的创建注册都在```configureAndRefreshWebApplicationContext()```这个方法完成：  
```
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
        //此处如无特殊指定一般情况下会相等
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// 一般情况下是没有特殊指定的，所以会生成默认id，org.springframework.web.context.WebApplicationContext:+项目的ContextPath
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath()));
			}
		}
        //contextConfigLocation，此处加载各种配置文件
		wac.setServletContext(sc);
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// 此处实现返回StandardServletEnvironment.class，也就是ConfigurableWebEnvironment的子类，初始化PropertySources
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		customizeContext(sc, wac);
        //上下文刷新
		wac.refresh();
	}
```
而```wac.refresh()```这个方法就是上下文刷新的所有的实现所在：  
```
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 上下文刷新前准备.
			prepareRefresh();

			// 初始化beanFactory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 填充beanFactory.
			prepareBeanFactory(beanFactory);

			try {
				// 允许在上下文子类中对bean工厂进行后处理.
				postProcessBeanFactory(beanFactory);

				// 激活各种beanFactory的处理器.
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册拦截bean创建的bean处理器.
				registerBeanPostProcessors(beanFactory);

				// 为上下文初始化message,即不同语言的消息体，国际化处理.
				initMessageSource();

				// 初始化此上下文的事件广播器.
				initApplicationEventMulticaster();

				// 子类初始化其他特殊bean.
				onRefresh();

				// 注册监听器.
				registerListeners();

				// 初始化所有其他单例bean.
				finishBeanFactoryInitialization(beanFactory);

				// 初始化完成，发布事件.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// 异常则销毁所有bean.
				destroyBeans();

				// 重置 'active' 状态.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// 重置缓存，因为不需要再次初始化了
				resetCommonCaches();
			}
		}
	}
```
##### prepareRefresh
整个过程中```prepareRefresh()```完成了刷新前的准备工作，核心代码如下：  
```
	protected void prepareRefresh() {
		// 切换active状态.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);
		....
		//由子类实现，初始化propertySource，其实现是MutablePropertySources
		//其内里实现是一个COWList，private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();
		initPropertySources();
		//验证属性合法性，一般情况下可以继承对应ApplicationContext类添加
		getEnvironment().validateRequiredProperties();
		....
	}
```
举例添加校验属性：  
```
public class CustomApplicationContext extends XmlWebApplicationContext {

    @Override
    protected void initPropertySources() {
        super.initPropertySources();
        //把"MYSQL_HOST"作为启动的时候必须验证的环境变量
        getEnvironment().setRequiredProperties("MYSQL_HOST");
    }
}
```
```initPropertySources()```初始化的是我们的所有的配置信息，初始化有严格的顺序，如果一个key在第一个```PropertySource```被找到，那么之后其他```PropertySource```的同名key会被忽略，所以我们要实现自己的配置中心就是要将我们的```PropertySource```放在首位：  
```
	protected void initPropertySources() {
		//此处env类型为StandardServletEnvironment
		ConfigurableEnvironment env = getEnvironment();
		//env.propertySourceList就是PropertySource属性列表
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
```
##### obtainFreshBeanFactory
```obtainFreshBeanFactory()```负责初始化beanFactory，整体比较简单，具体实现逻辑在其所调用的两个方法里，其中```refreshBeanFactory()```负责刷新beanFactory，其实现逻辑是如果当前存在beanFactory，那么就销毁所有单例bean，并销毁当前beanFactory并重新重建beanFactory：  
```
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
```
factoryBean刷新方法如下：  
```
	protected final void refreshBeanFactory() throws BeansException {
		//判断beanFactory!=null
		if (hasBeanFactory()) {
			destroyBeans();//销毁所有bean
			closeBeanFactory();//销毁判断beanFactory
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);//设置各种beanProcessor，environment，
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```
```loadBeanDefinitions()```是一个模板方法，具体由子类实现，我们以最常用的```AnnotationConfigWebApplicationContext```为例：
```
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
		//注册各种beanProcessor，如@Value，@Autowired，@Resource，@Bean，@Configuration，@EventListener
		AnnotatedBeanDefinitionReader reader = getAnnotatedBeanDefinitionReader(beanFactory);
		//扫描Component，Controller，Indexed，Repository，Service
		ClassPathBeanDefinitionScanner scanner = getClassPathBeanDefinitionScanner(beanFactory);

		BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
		if (beanNameGenerator != null) {
			reader.setBeanNameGenerator(beanNameGenerator);
			scanner.setBeanNameGenerator(beanNameGenerator);
			beanFactory.registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
		}

		ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
		if (scopeMetadataResolver != null) {
			reader.setScopeMetadataResolver(scopeMetadataResolver);
			scanner.setScopeMetadataResolver(scopeMetadataResolver);
		}

		if (!this.componentClasses.isEmpty()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Registering component classes: [" +
						StringUtils.collectionToCommaDelimitedString(this.componentClasses) + "]");
			}
			reader.register(ClassUtils.toClassArray(this.componentClasses));
		}

		if (!this.basePackages.isEmpty()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Scanning base packages: [" +
						StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
			}
			scanner.scan(StringUtils.toStringArray(this.basePackages));
		}

		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				try {
					Class<?> clazz = ClassUtils.forName(configLocation, getClassLoader());
					if (logger.isTraceEnabled()) {
						logger.trace("Registering [" + configLocation + "]");
					}
					reader.register(clazz);
				}
				catch (ClassNotFoundException ex) {
					if (logger.isTraceEnabled()) {
						logger.trace("Could not load class for config location [" + configLocation +
								"] - trying package scan. " + ex);
					}
					int count = scanner.scan(configLocation);
					if (count == 0 && logger.isDebugEnabled()) {
						logger.debug("No component classes found for specified class/package [" + configLocation + "]");
					}
				}
			}
		}
	}
```
##### postProcessBeanFactory
```postProcessBeanFactory()```主要是填充```beanFactory```:  
```
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 设置类加载器，el解析器，属性注册解析器
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 将当前contex设置到beanProcessor.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		//设置需要忽略的自动装配依赖接口
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```