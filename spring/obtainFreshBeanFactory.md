#### obtainFreshBeanFactory
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
##### 1、loadBeanDefinitions
```loadBeanDefinitions(DefaultListableBeanFactory)```是一个模板方法，具体由子类实现，我们以最常用的```XmlWebApplicationContext```为例：
```
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
		// 创建一个xmlReader.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
```
```loadBeanDefinitions(XmlBeanDefinitionReader)```主要是填充读取配置文件并解析web.xml配置的spring配置文件，如```spring-servlet.xml```:  
```
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}
```
##### 2、XmlBeanDefinitionReader.loadBeanDefinitions(String)
```XmlBeanDefinitionReader.loadBeanDefinitions(String)```加载配置的xml文件:
```
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}
        //XmlWebApplicationContext是ResourcePatternResolver的子类，所以会走到这里，不过核心方法都是loadBeanDefinitions(EncodedResource)
		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
                //这里虽然传值是一个集合，但是内部是一个循环调用单个Resource的方法
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
```
```loadBeanDefinitions(EncodedResource)```开始读取xml配置文件并解析为beanDefinition：
```
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}
        //此处是一个HashSet，列表的内容是所有已处理过的Resource，作用是检测配置文件的循环引用，如config-a.xml import config-b.xml;config-b.xml import config-a.xml
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}

		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
            //解析xml，注册beanDefinition
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
```
##### 3、doLoadBeanDefinitions(InputSource, Resource)
```doLoadBeanDefinitions(InputSource, Resource)```将制定文件加载为Document对象，解析并注册BeanDefinition：
```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
            //将resource读取为Document对象
			Document doc = doLoadDocument(inputSource, resource);
            //从Document对象注册BeanDefinition
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
```
```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        //生成一个DefaultBeanDefinitionDocumentReader对象
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
        //往下调用
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```
##### 4、DefaultBeanDefinitionDocumentReader
```
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
```
```
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
        //创建新的BeanDefinitionParserDelegate对象，并从parent进行初始化
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
        //判断是否默认命名空间，默认命名空间为"http://www.springframework.org/schema/beans",如果是默认命名空间
		if (this.delegate.isDefaultNamespace(root)) {
            //获取beans标签的profile属性
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
                //如果配置了profile，那么判断配置文件的profile是否是启用的profile，或者是否是禁用的profile，
                //如spring.profiles.active=prod,那么profile=dev的配置文件就不加载，或者spring.profiles.active=default，那么profile=default的配置文件也加载
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
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
```
这里判断是否默认命名空间决定走那个方法，默认命名空间的默认标签走默认标签加载，否则走自定义标签加载方法
```
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        //判断命名空间是否默认命名空间，也就是beans
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
                    //判断标签是否默认命名空间，默认命名空间标签为beans、bean、import、alias
					if (delegate.isDefaultNamespace(ele)) {
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
parseDefaultElement()会根据标签类型判断需要走哪个方法，import标签加载指定配置文件最终还是到```parseBeanDefinitions```方法，beans标签会执行```doRegisterBeanDefinitions```方法，所以这个方法里重要的部分是```processBeanDefinition```方法
```
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        //import标签加载引入的配置文件，再按照配置文件解析流程执行,DefaultBeanDefinitionDocumentReader.importBeanDefinitionResource
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
        //alias标签为指定beanName设置别名
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
        //bean标签加载指定bean,重点关注
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
        //beans标签执行doRegisterBeanDefinitions方法，DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```
首先看一下bean标签的加载，也就是```processBeanDefinition```方法：
```
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //获取BeanDefinitionHolder，也就是解析xml的bean标签，获取标签的各种属性，如id，class等
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
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
从三个部分看processBeanDefinition，前两部分都和个部分是```BeanDefinitionParserDelegate```有关，首先是```parseBeanDefinitionElement```,也就是对bean标签的解析：
```
    @Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}

	@Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
        //获取标签的id属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
        //获取标签的name属性
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
        //根据,; 三个符号分割name属性，将他们放到alias集合
		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}
        //如果id属性是空的，且name不为空，则取分割后的第一个name作为beanName，且将第一个beanName移除
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isTraceEnabled()) {
				logger.trace("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}
        //从上边的调用，这一步的containingBean为null，所以肯定会执行checkNameUniqueness方法，checkNameUniqueness的作用是检测beanName是否已经存在，首先检测beanName是否已存在，如果不存在则检测aliases是否已存在，如果都不存在就添加到usedNames列表，如果有一项存在，则抛出BeanDefinitionParsingException异常
		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}
        //这一步获取bean标签的各种属性，构造一个beanDefinition
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
        //如果beanDefinition构造成功且beanName为空，name就根据beanDefinition构造一个beanName
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isTraceEnabled()) {
						logger.trace("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
    
    @Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
        //获取class属性的值
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
        //获取parent属性的值
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
            //创建BeanDefinition对象，设置parentName，如果classLoader不为空，则通过readerContext的classLoader获取beanClass，否则设置beanClassName
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
            //设置BeanDefinition的属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            //设置bean的描述信息，取bean标签下的description子标签
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
            //设置bean的元信息,取bean标签下的meta子标签
			parseMetaElements(ele, bd);
            //设置bean的lookup-method信息，去bean标签下的lookup-method标签，主要用来设置某个方法的返回是另外一个声明的bean，可以用来替代ApplicationContextAware，，可以用来在单例bean获取非单例bean的实例
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

    public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
        //如果标签有属性singleton，就抛出异常，提示使用scope属性
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
        //如果标签有属性scope，就设置bean的scope，scope有singleton、prototype、request、session、global session
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
        //如果containingBean不为空，则获取containingBean的scope，上边过来的是null
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.
			bd.setScope(containingBean.getScope());
		}
        //如果标签有属性abstract，则设置是否抽象bean，抽象bean一般用作公共父bean，如两个类都需要注入dataSource，则可以将DataSource提到抽象bean，抽象bean不是真实存在的bean
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}
        //如果标签有属性lazy-init，首先判断是否是默认值default或者空，如果是则取beans标签的default-lazy-init,这个值是在构建delegate对象时初始化的
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (isDefaultValue(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));
        //获取标签的autowire属性，获取注入类型，这一步和lazy-init标签一样，也会判断默认值，如果是则取beans标签的default-autowire属性，然后判断注入类型，如byName、byType、constructor或者autodetect
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));
        //如果标签有depends-on属性，则获取depends-on属性，根据,;分割成list设置bena的dependsOn，dependsOn用来控制bean的加载顺序，如果beanA dependsOn beanB，那么beanB会比beanA先初始化
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}
        //如果标签有autowire-candidate属性，如果是默认值，则取default-autowire-candidate,分割成数组然后匹配beanName，否则判断标签的值是否是true，如果设置为false则这个bean不作为候选bean，用在注入方式是byType时，一个接口有两个实现的类，可以为一个设置为不作为候选bean，否则会报NoUniqueBenaDefinitionException
		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if (isDefaultValue(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}
        //如果标签有primary属性，设置bean的primary属性，primary用来指定主要bean，如果一个接口有多个实现的类，有限注入primary=true的实现
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}
        //如果标签有init-method属性，设置bean的initMethodName属性，如果为空则且beans标签的default-init-method不为空，则设置beans标签的属性，init-method主要是用来bean初始化时执行的方法，用来替代继承 InitializingBean接口
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}
        //如果标签有destoryMethod属性，设置bean的destroyMethod属性，如果为空且default-destroy-method不为空，则设置beans标签的属性，destory-method主要是用来bean销毁时执行的方法
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}
        //factoryMethod和factoryBean标签结合使用，用来设置bean的初始化工长类和方法
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```