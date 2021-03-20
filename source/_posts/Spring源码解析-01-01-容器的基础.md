---
title: Spring源码解析-01-01-容器的基础
copyright: true
comments: true
related_posts: true
date: 2021-03-11 23:23:29
tags: 容器的基础
categories: Sprig源码解析
---



# 容器的基础（Bean的解析）

XmlBeanFactory： 

## 配置文件的封装：

- 文件资源统一抽象为： InputStreamSource 资源, Resource 继承 InputStreamSource，

- 然后 通过Resource资源，可以 拿到InputStream，

xmlBeanFactory的构造函数：

```java
	/**
	 * Create a new XmlBeanFactory with the given resource,
	 * 创建一个xml
	 * which must be parsable using DOM.
	 * @param resource the XML resource to load bean definitions from
	 * @throws BeansException in case of loading or parsing errors
	 */
	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}

/**
	 * Create a new XmlBeanFactory with the given input stream,
	 * which must be parsable using DOM.
	 * @param resource the XML resource to load bean definitions from
	 * @param parentBeanFactory parent bean factory
	 * @throws BeansException in case of loading or parsing errors
	 */
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
        // 资源加载的真正实现
		this.reader.loadBeanDefinitions(resource);
	}
```



## 加载xmlBeanDefinitions

- 在 XmlBeanFactory中 调用了 super(parentBeanFactory); 一直往上追可以看到 ,这段代码作用是：

  - 忽略给定接口的自动装配功能

- 这样的目的是为什么呢，效果如何：

  - 举例子来说：当A中有属性B，那么当Spring在获取A的Bean的时候，如果其属性B还没有初始化，那么Spring会自动初始化B，这也是Sprnig  提供的一种特性，

  - 但是某些情况下，B不会被初始化，其中的一种情况就是 B实现了BeanNameAware 接口。
  - spring是这么介绍的： 自动装配时 忽略给定的依赖接口，典型应用就是 通过其他方式解析 Application 上下文注册依赖，类似于 BeanFactory 通过BeanFactoryAware 进行注入 或者ApplicationContext通过ApplicationContextAware 进行注入

```java
	/**
	 * Create a new AbstractAutowireCapableBeanFactory.
	 */
	public AbstractAutowireCapableBeanFactory() {
		//
		super();
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
	}
```

### loadBeanDefinitions

```java
/**
	 * Load bean definitions from the specified XML file.
	 * @param encodedResource the resource descriptor for the XML file,
	 * allowing to specify an encoding to use for parsing the file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 */
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}
		// 1.封装资源文件 ，首先 对参数Resource 使用 EncodedResource
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		// 2. 获得输入流
		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			// 3.重点方法
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

### 获取Document

- xmlBeanFactoryReader 类对于文档并没有亲力亲为，而是委托 给了 DocumentLoader 去执行，这力的 DocumentLoader是个接口，而真正的调用的是 ： DefaultDocumentLoader 

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
			Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}catch (Exception ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
```

### 解析并注册BeanDefinitions

- 当把文件转换为Document之后，接下来的提取以及注册 bean。
- 这个方法很好的应用了 面向对象中单一职责的原则，将逻辑处理委托给单一的类 进行处理，而这个逻辑处理类就是 BeanDefinitionDocumentReader
- BeanDefinitionDocumentReader 是一个接口，而实例化的工作是在 createBeanDefinitionDocumentReader（）中完成的，BeanDefinitionDocumentReader的最终类型是 DefaultBeanDefinitionDocumentReader
- DefaultBeanDefinitionDocumentReader::doRegisterBeanDefinitions() 提取root，以便将root作为参数继续 BeanDefinition 的注册

```java
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		//使用 DefaultBeanDefinitionDocumentReader 实例化 BeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 将环境变量设置其中
		// 记录统计前BeanDefinition 的加载个数
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 加载以及注册 bean
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 记录本次加载 的BeanDefinition 的个数
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

### 核心方法 registerBeanDefinitions

- 对profile 处理，然后开始解析 
- 跟进  preProcessXml(),postProcessXml(root);发现代码是空的，为啥是空的呢

**面向对象常说的一句话： 一个类要么是面向继承的设计，要么就用final 修饰。因为 DefaultBeanDefinitionDocumentReader 不是final类，所以他是面向继承设计。** 

**这2个方法是为子类设计的 ，这个模板方法模式，如果继承自 DefaultBeanDefinitionDocumentReader 的子类需要在Bean解析前后 做一些处理的话，那么只需要 重写这2个方法。**

```java
protected void doRegisterBeanDefinitions(Element root) {
		// 专门处理解析
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			//. 处理 profile 属性
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
		// 3. 解析前处理，留给子类实现
		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		// 4. 解析后处理，留给子类实现
		postProcessXml(root);

		this.delegate = parent;
	}
```



### parseBeanDefinitions(root, this.delegate)

- 如何解析XML 的元素就跳过了，包含自定义标签之类的 。感觉也用不到，毕竟现在大部分都是注解开发 ，就先不看了,太繁琐了