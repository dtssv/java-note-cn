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
		//获取web.xml配置contextConfigLocation
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
				// 后置处理beanFactory.
				postProcessBeanFactory(beanFactory);

				// 执行各种beanFactory的处理器.
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

