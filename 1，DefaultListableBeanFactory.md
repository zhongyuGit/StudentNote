1，DefaultListableBeanFactory

的结构图

![1587793060371](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587793060371.png)

各个类的作参考课本的第25页

DefaultListableBeanFactory主要是针对Bean注册后的处理。

XMLBeanFactory 继承了DefaultListableBeanFactory,并且进行的拓展，主要是从XML文档中读取BeanDefinition,也就是Bean的定义，对于注册及获取bean是使用DefaultListableBeanFactory继承的方法去实现，唯独不同的是XMLBeanFactory 增加了XmlBeanDefinitionReader类型的reader属性，在Xmlbeanfactory钟主要使用reader属性来进行对资源文件的读取和注册。

2.XmlBeanDefinitionReader做的主要工作：

![1587794401460](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587794401460.png)



ResourceLoader将资源文件路劲转换为对应的Resource文件![1587795071871](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587795071871.png)，

**XmlBeanDefinitionReader做的主要工作：**



1.使用继承的AbstractBeanDefinitionReader的是实现接口BeanDefinitionReader来实现资源文件到BeanDefinition的各个功能

通过DocumentLoader将Resource文件转为Document对象

![1587795381785](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587795381785.png)

BeanDefinitionDocumentReader将document对象进行解析![1587794992207](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587794992207.png)BeanDefinitionParserDelegate定义了解析Element的各种方法

![1587795773360](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587795773360.png)

真正启动：

BeanFactory facotry = new XmlBeanFactory((也就是Resource)new ClasspathResource("xxxx.xml"));

我们看XmlbeanFactory的构造方法![1587800360487](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587800360487.png)

```java
this.reader.loadBeanDefinitions(resource);
```

这个就是我们private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);真正加载数据的的点。

但是前面有个super(BeanFactory parentBeanfactory),追溯到AbstractAutowireCapableBeanfactory中

![1587800566161](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587800566161.png)

ignoreDepedencyInterface(Object o),主要的功能是忽略给定接口的自动装配功能，

当a中有b属性的时候，spring在获取a的bean的时候如果其属性b还没有初始化，那么spring会自动初始化b。当时某些情况b不会被初始化，其中一种就是b实现了上面的三个接口。spring的介绍：自动装配时候忽略给定的依赖接口，典型应用就是通过其他解析方式application上下文注册依赖，类似于BeanFactory通过BeanFactoryAware进行注入或者是ApplicationContext通过ApplicationContextAwaer进行注入。

- [ ] this.reader.loadBeanDefinitions(resource);//进入处理

  ```java
  @Overridepublic int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {   
      return loadBeanDefinitions(new EncodedResource(resource));
  }
  ```

EncodedResource的作用是用于对资源文件的编码处理，主要的逻辑体现在getReader这里，

```java
	public Reader getReader() throws IOException {
		if (this.charset != null) {
			return new InputStreamReader(this.resource.getInputStream(), this.charset);
		}
		else if (this.encoding != null) {
			return new InputStreamReader(this.resource.getInputStream(), this.encoding);
		}
		else {
			return new InputStreamReader(this.resource.getInputStream());
		}
	}
//意思是，当你不设置字符集和编码的时候，默认的返回值就是你传给EncodeResource的Resource,这里是有个编码处理的类。
```

![1587803522509](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1587803522509.png)

EncodeResource的构造方法，默认字符集和编码是null.

**loadBeanDefinitions（EncodeResource）**



```java
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}
        //通过属性来记录已经加载的资源

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}

		try (
            //从EncodeResource中获取封装好的Resource对象，且从中获得输入流
            InputStream inputStream = encodedResource.getResource().getInputStream()) {
            //这个类不属于spring，他的路径是org.xml.sax.InputResource
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}//进入核心的逻辑
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

回顾一下，我们已经获得xml文件对应的Resource,然后对Resource进行编码处理的EncodeResource(考虑了编码问题的举措) ,通过sax读取xml文件的方式来准备ImportSource对象，最后将准备的对象传入doLoadBeanDefinitions(inputSource, encodedResource.getResource())真正处理的地方。

```java
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
            //1.里面还有获取对XML文件的验证模式，加载xml获得对应的document,
			Document doc = doLoadDocument(inputSource, resource);
            //2.根据得到的document注册bean的信息
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

分析一下上面的步骤，我定义的步骤是支撑着spring容器部分的实现基础，尤其是bean的注册，逻辑很复杂

首先获得文档要获得xml文件的验证模式，xml文件的验证模式保证了xml文件的正确性，而比较常用的验证模式有两种，DTD和XSD,

DTD(Document Type Definition)文档类型定义，（百度后面补充），两者的区别



```java
	protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
```

> ```java
> 	protected int getValidationModeForResource(Resource resource) {
> 		int validationModeToUse = getValidationMode();//默认得到private int validationMode = VALIDATION_AUTO;
>         //如果没有手动指定就使用默认的自动检测
> 		if (validationModeToUse != VALIDATION_AUTO) {
> 			return validationModeToUse;
> 		}
> 		int detectedMode = detectValidationMode(resource);//
> 		if (detectedMode != VALIDATION_AUTO) {
> 			return detectedMode;
> 		}
> 		return VALIDATION_XSD;
> 	}
> ```
>

 

```java
	protected int detectValidationMode(Resource resource) {
		if (resource.isOpen()) {
			throw new BeanDefinitionStoreException(
					"Passed-in Resource [" + resource + "] contains an open stream: " +
					"cannot determine validation mode automatically. Either pass in a Resource " +
					"that is able to create fresh streams, or explicitly specify the validationMode " +
					"on your XmlBeanDefinitionReader instance.");
		}

		InputStream inputStream;
		try {
			inputStream = resource.getInputStream();
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Unable to determine validation mode for [" + resource + "]: cannot open InputStream. " +
					"Did you attempt to load directly from a SAX InputSource without specifying the " +
					"validationMode on your XmlBeanDefinitionReader instance?", ex);
		}

		try {
			return this.validationModeDetector.detectValidationMode(inputStream);//专门处理类
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("Unable to determine validation mode for [" +
					resource + "]: an error occurred whilst reading from the InputStream.", ex);
		}
	}
```

在detectValidationMode中，又将自动检测验证模式委托给了相关的处理类XmlValidationModeDetector，调用了XmlValidationModeDetector的detectValidationMode方法

```java
	public int detectValidationMode(InputStream inputStream) throws IOException {
		// Peek into the file to look for DOCTYPE.
		BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
		try {
			boolean isDtdValidated = false;
			String content;
			while ((content = reader.readLine()) != null) {
				content = consumeCommentTokens(content);
                //content<!
                //如果读取的行是空或者是注释则略过
				if (this.inComment || !StringUtils.hasText(content)) {
					continue;
				}
				if (hasDoctype(content)) {
					isDtdValidated = true;
					break;
				}
                //读到<开始符号，验证模式一定会在开始符号之前
				if (hasOpeningTag(content)) {
					// End of meaningful data...
					break;
				}
			}
			return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
		}
		catch (CharConversionException ex) {
			// Choked on some character encoding...
			// Leave the decision up to the caller.
			return VALIDATION_AUTO;
		}
		finally {
			reader.close();
		}
	}

```

spring用来检测验证模式的方法是判断是否包含doctype,包含则是dtd,否则就是xsd.

继续，经过验证模式的准备步骤就可以进行document加载，同样xmlbeandefinitionReader对于读取文档并没有自己来，委托给了DocumentLoader去执行，这里的document是个接口，真正调用的是**DefaultDocumentLoader**

```java
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isTraceEnabled()) {
			logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		return builder.parse(inputSource);
	}
```



通过SAX解析xml文档都差不多，首先是创建DocumentBuilderFactory,再通过DocumentBuilderFactory创建DocumentBuilder,进而解析inpustSource来获取返回的Document对象

**对**于public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,**
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware)中的参数entityResolver，传入的值的是XmlBeanDefinitionReader中的getEntityResolver()方法

```java
	protected EntityResolver getEntityResolver() {
		if (this.entityResolver == null) {
            //
			// Determine default EntityResolver to use.
			ResourceLoader resourceLoader = getResourceLoader();
			if (resourceLoader != null) {
				this.entityResolver = new ResourceEntityResolver(resourceLoader);
			}
			else {
				this.entityResolver = new DelegatingEntityResolver(getBeanClassLoader());
			}
		}
		return this.entityResolver;
	}
```

EntityResolver

如果Sax应用程序想要处理外部实体，则必须实现改接口，且使用setEntityResolver方法向sax驱动器注册一个实例。

**1 解析和注册BeanDefinitions**

获得了document后，接下来的提取及注册bean就开始了，继续上面分析，拿到文件的document实例之后，

```java
		Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);//进去
```

进入registerBeanDefinitions

```java
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        //使用DefaultBeanDefinitionDocument初始化BeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        //记录统计前BeanDefinition加载个数
		int countBefore = getRegistry().getBeanDefinitionCount();
        //加载及注册bean
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        //记录本次加载的beanDefinition的个数
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

 BeanDefinitionDocumentReader是一个接口，而工作类是DefaultBeanDefinitionDocumentReader

```java
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
```

```java
	protected void doRegisterBeanDefinitions(Element root) {


		if (this.delegate.isDefaultNamespace(root)) {
            //处理profile属性
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
        //调换了一下下面两行在源码的顺序
        //专门解析处理
        BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
		//解析前的处理，留给继承该类的子类实现，如果需要在解析前后做一些处理
		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
        //解析后的处理，留给继承该类的子类实现
		postProcessXml(root);

		this.delegate = parent;
	}
```

profile是可以在配置文件中部署开发环境和生产环境，可以在web文件中，指定spring.profile.active,如果spring配置文件中又配置profile，就回到环境变量中去寻找。具体参考百度。

2.解析并注册BeanDefinition

```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                        //对bean的处理
						parseDefaultElement(ele, delegate);
					}
					else {
                         //对bean的处理
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

spring配置文件里有两大类的Bean声明，

第一种就是默认的<bean id=''' class=''>

第二种是自定义的，比如<tx:annotion-driven/>

对于两种方式读取的方法及解析的差别是巨大的，如果是默认的配置，spring知道怎么做，如果是自定义的，spring就需要用户实现一些接口和配置。对于根节点或者是子节点如果是采用默认的命名空间则采用parseDefaultElement（）解析，否则使用delegate.parseCustomElement(ele);解析。而判断是否是默认的命名空间还是自定义的命名空间的反法则是使用node.getNameSpaceURI()获取命名空间，并与spring中固定的命名空间http://www.SpringFramework.org/schema/beans进行比对，如果一致就是默认，反之则反。



# **第三章 默认标签的解析**

```java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        //这是对import标签的处理
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
        //这是对alias标签的处理
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
        //这是对bean标签的处理
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
        //这是对beans标签的处理
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

我们这里首先对最重要的bean标签进行解析，这是最为复杂和重要的,进入processBeanDefinition(ele, delegate);

```java
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
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

大体的逻辑：

1.首先委托BeanDefinitionParserDelegate类的parseBeanDefinitionElement进行元素解析，返回BeanDefinitionHolder类型的实例bdHolder,进过这个反法之后，bdHolder已经包含我们的配置文件中的配置的各种属性，例如class,id,name,alias之类的属性。

2.当放回的bdholder不为空的情况下若存在默认标签的子节点下再有自定义属性，还需要在再次对自定义标签进行解析。

3.解析完成后，需要对bdHolder进行注册，同样，注册操作委托BeanDefinitionReaderUtils的registerBeanDefinition方法。

4.最后发出响应事件，通知相关的监听器，这个bean的加载已经完成。

我们一步一步来解析

**BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);**

```java
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
        //id属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
        //name属性
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		//分割name属性
		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isTraceEnabled()) {
				logger.trace("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}

		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
//如果beanName为空时，那么就根据spring提供的命名规则为当前bean生成对应的beanName(即类似文件中id)
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
```

以上就是对默认标签的解析，尽管我们先只是看到了id和name的解析，但是思路我么已经了解了。在开始对属性展开全面的解析前，spring在外层又做了一个当前层的功能框架，在当前层的主要工作内容包括：

1.提取bean的id和name属性

2.进一步解析其他所有属性并统一封装到 GenericBeanDefinition（默认的）类型的实例

3.如果检查到bean没有指定beanName，那么就使用默认规则生成beanName

4.将获取到的信息封装到BeanDefinitionHolder中

我们进一步查看步骤2的其他属性的解析

**AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);**

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));
		//解析class属性
		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
    	//解析parent属性
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
            //创建用于GenericBeanDefinition bd = new GenericBeanDefinition();
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			//硬编码解析默认bean的各种属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
			//解析元数据
			parseMetaElements(ele, bd);
            //解析lookup-method属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            //解析replace-method属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
			//解析构造方法参数
			parseConstructorArgElements(ele, bd);
            //解析property子元素
			parsePropertyElements(ele, bd);
            //解析qualifier子元素
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

