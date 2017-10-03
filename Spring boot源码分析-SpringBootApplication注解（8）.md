# Spring boot源码分析-SpringBootApplication注解（8）
我们一定很奇怪，调用	SpringApplication.run(ChapterProfilesApplication.class, args);的代码是怎么启动spring并加载所有的bean的，其实关键就是在SpringBootApplication注解，今天我们就来讲讲这个注解

*   先看这个注解的源码,主要的组成有@SpringBootConfiguration，@EnableAutoConfiguration，@ComponentScan，scanBasePackages() ，scanBasePackageClasses（）这些参数
  
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class))
public @interface SpringBootApplication {
  
	Class<?>[] exclude() default {};
   
	String[] excludeName() default {};
  
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}
```
*   然后再查看@SpringBootConfiguration注解其实就是对@Configuration的包装  我们可以认为是@Configuration注解

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```
*   @EnableAutoConfiguration注解其实包含了两个Import的类  一个是
EnableAutoConfigurationImportSelector.class还有一个是AutoConfigurationPackages.Registrar.class

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

*   然后再查看@ComponentScan(excludeFilters = @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class))
  就是增加了ComponentScan注解
  
*   所以总结来说  @SpringBootConfiguration注解就是由一下注解来组合而成的
  1.@Configuration
  2.@ComponentScan
  3.EnableAutoConfigurationImportSelector.class
  4.AutoConfigurationPackages.Registrar.class

### @Configuration的作用
 从spring 3.0开始支持javaConfig的方式进行配置，具体表现在spring可以 通过java类当做配置文件进行处理，从而使代码摆脱大段的配置文件的困扰
 
 在springboot中有BeanDefinitionLoader，就是我们前面讲过的BeanDefinitionReader外观模式，我们上篇文章讲了，当使用javaConfig的方式进行配置的时候使用的是AnnotatedBeanDefinitionReader，调用了register方法
 
```
  //新建AnnotatedBeanDefinitionReader
	this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
  //注册bean信息
	this.annotatedReader.register(source);
```
*   我们查看上面的注册方法发现最终调用的事registerBean，我们可以看到他通过class进行bean注册的全过程

```
	public void register(Class<?>... annotatedClasses) {
		for (Class<?> annotatedClass : annotatedClasses) {
			registerBean(annotatedClass);
		}
	}

	public void registerBean(Class<?> annotatedClass) {
		registerBean(annotatedClass, null, (Class<? extends Annotation>[]) null);
	}
```

```
	@SuppressWarnings("unchecked")
	public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
		//支持元数据的beandefinition一般实现
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		//this is new StandardAnnotationMetadata(beanClass, true);
		//注解判断  如果有注解元数据并且注解元数据还没满足条件  返回true  那么就不注册这个配置文件了
		//这里好像说的是   配置类可以写几个一样的   在不同的环境上进行原型   传入统一参数就好了
		//abd.getMetadata()  标准封装的注解元素
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}
		//获取封装完成的scopeMetadata  有可能没有这个注解  直接回来标准的
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName());
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		//创建ScopedProxyMode代理
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
```
*   其中新建AnnotatedBeanDefinitionReader的动作中，构造函数中有依据很重要的代码AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
这句代码想spring容器中注册了ConfigurationClassPostProcessor.class类，这个类容器调用refresh()方法，也就是springboot调用run()方法里面refreshContext(context);以后会调用invokeBeanFactoryPostProcessors，从而初始化所有的bean（有兴趣的可以去看spring整个加载过程和生命周期）

```
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		//Conditional注解决定器
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		//这个时关键   注册AnnotationConfigProcessor
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```
*   而作为javaConfig的参数类会被作为AnnotatedGenericBeanDefinition注册到BeanFactory容器中，下面是AnnotatedGenericBeanDefinition的结构

![](http://osgqa4bwf.bkt.clouddn.com/2017-10-03-15061540753497.jpg)
*   首先把配置Class封装成AnnotatedGenericBeanDefinition，我们看这个类的作用。

    1.首先他是一个GenericBeanDefinition意味着这个类是一个标准的Bean定义类，然后在下面进行了bean的注册，BeanFactory容器中最终也是会有这个单例的bean的
    2.这个类在初始化的时候解析了类上所有的注解放入元数据中
```
	this.metadata = new StandardAnnotationMetadata(beanClass, true);
	
	
	public StandardAnnotationMetadata(Class<?> introspectedClass, boolean nestedAnnotationsAsMap) {
		super(introspectedClass);
		this.annotations = introspectedClass.getAnnotations();
		this.nestedAnnotationsAsMap = nestedAnnotationsAsMap;
	}

```
*   我们再看ConfigurationClassPostProcessor.class的方法postProcessBeanFactory，在这个里面就是spring通过注解的方式解析所有的bean并注册的过程，有兴趣可以去看看spring的代码

```
@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		if (!this.registriesPostProcessed.contains(factoryId)) {
			// BeanDefinitionRegistryPostProcessor hook apparently not supported...
			// Simply call processConfigurationClasses lazily at this point then.
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}
		enhanceConfigurationClasses(beanFactory);
	}
```

```
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		//候选配置类的List数组
		List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
					ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
				if (logger.isDebugEnabled()) {
					//bean以及处理过了   说明上面两个标记时处理的时候标记上去的
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		}
		
		// Sort by previously determined @Order value, if applicable
		// Configuration北标记的情况下会进行排序的
		Collections.sort(configCandidates, new Comparator<BeanDefinitionHolder>() {
			@Override
			public int compare(BeanDefinitionHolder bd1, BeanDefinitionHolder bd2) {
				int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
				int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
				return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
			}
		});
		
		// Detect any custom bean name generation strategy supplied through the enclosing application context
		SingletonBeanRegistry singletonRegistry = null;
		if (registry instanceof SingletonBeanRegistry) {
			singletonRegistry = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet && singletonRegistry.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {
				BeanNameGenerator generator = (BeanNameGenerator) singletonRegistry.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
				this.componentScanBeanNameGenerator = generator;
				this.importBeanNameGenerator = generator;
			}
		}
		
		// Parse each @Configuration class
		/*
		 * metadataReaderFactory = new CachingMetadataReaderFactory();
		 * problemReporter = new FailFastProblemReporter();
		 * environment
		 * resourceLoader = new DefaultResourceLoader();
		 * componentScanBeanNameGenerator = new AnnotationBeanNameGenerator();
		 * 配置解析委托处理类
		 * 
		 */
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);
		//配置文件集合
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());
		do {
			//解析配置
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				//这个类似于beandefinitionreader
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			//直接注册扫描出来的类  --解析类数据   这里进行所有bean的解析注册
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<String>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition beanDef = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(beanDef.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(beanDef, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (singletonRegistry != null) {
			if (!singletonRegistry.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
				singletonRegistry.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
			}
		}

		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}
```

### @ComponentScan的作用
也是在ConfigurationClassPostProcessor.class解析的时候遇到@ComponentScan注解，然后扫描注解配置下面的所有的类，然后进行注册bean

### EnableAutoConfigurationImportSelector.class的作用
ConfigurationClassPostProcessor.class这个注册切面，在调用
ConfigurationClassParser类的parser.parse(candidates)的时候，
获取EnableAutoConfiguration的配置  在spring-boot-autoconfigure工程下面的spring.factories。然后解析所有被扫描出来的Configraution

```
public void parse(Set<BeanDefinitionHolder> configCandidates) {
		this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {
				if (bd instanceof AnnotatedBeanDefinition) {
					parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
				}
				else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
					parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
				}
				else {
					parse(bd.getBeanClassName(), holder.getBeanName());
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
			}
		}

		processDeferredImportSelectors();
	}
```

```
private void processDeferredImportSelectors() {
		List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
		this.deferredImportSelectors = null;
		Collections.sort(deferredImports, DEFERRED_IMPORT_COMPARATOR);

		for (DeferredImportSelectorHolder deferredImport : deferredImports) {
			ConfigurationClass configClass = deferredImport.getConfigurationClass();
			try {
				String[] imports = deferredImport.getImportSelector().selectImports(configClass.getMetadata());
				processImports(configClass, asSourceClass(configClass), asSourceClasses(imports), false);
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to process import candidates for configuration class [" +
						configClass.getMetadata().getClassName() + "]", ex);
			}
		}
	}
```

所以，只要在spring.factories进行声明，EnableAutoConfiguration的扫描类，就可以开箱即用相应的功能


### AutoConfigurationPackages.Registrar.class的作用
支持Java配置扩展的关键点就是@Import注解，Spring 3.0提供了这个注解用来支持在Configuration类中引入其它的配置类，包括ImportSelector和ImportBeanDefinitionRegistrar的实现类。


