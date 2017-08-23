# Spring boot源码分析-AnnotationConfigApplicationContext非web环境下的启动容器（2）

首先我们看容器的类图
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-15031299205209.jpg)
1.首先该类间接继承了AbstractApplicationContext（Spring容器最重要的抽象类，就有了容器最终要的一些功能）
2.该类还实现了AnnotationConfigRegistry  注解扫描注册接口  就是基于注解的容器   实现了读取spring注解加载容器的功能


### 容器启动，构造方法

-  首先我们看类的构造方法，主要做了以下几个工作
1.  AnnotatedBeanDefinitionReader BeanDefinition解析器用来解析带注解的bean
2.  ClassPathBeanDefinitionScanner  bean的扫描器  用来扫描类
3.  注册解析传入的配置类（使用类配置的方式进行解析）
4. 调用容器的refresh方法初始化容器

```
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
	private final AnnotatedBeanDefinitionReader reader;
	private final ClassPathBeanDefinitionScanner scanner;
	
	public AnnotationConfigApplicationContext() {
		//关键代码  里面有注解的BeanDefinitionreader
		this.reader = new AnnotatedBeanDefinitionReader(this);
		//bean扫描器
		this.scanner = new ClassPathBeanDefinitionScanner(this);
		logger.info(this.getBeanFactory());
	}
	
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		//关键代码   注册配置的类对象   （当配置类上有@Conditional注解且为matches方法返回false的时候  好像这个类就不注册了）
		register(annotatedClasses);
		refresh();
	}
}
```

-   在看他的父类的构造器  创建了DefaultListableBeanFactory  bean工厂
 
```
	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
```

-   我们现在对于容器的类型和大致的执行过程有了一个了解   知道容器启动的时候做了哪几个大的工作


### AnnotatedBeanDefinitionReader Bean解析器的分析

-   先看构造的源码

```
this.reader = new AnnotatedBeanDefinitionReader(this);
```
```
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}
	
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		//Conditional注解评估器
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		//这个时关键   注册AnnotationConfigProcessor
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```

-   注解的条件判断器ConditionEvaluator 该方法在初始化的时调用，当配置的类上有@Conditional注解并且返回false的时候  容器就不处理该类


AnnotationConfigApplicationContext

```
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		······
		register(annotatedClasses);
		······
	}
}

public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
}
```

AnnotatedBeanDefinitionReader

```
public void register(Class<?>... annotatedClasses) {
		for (Class<?> annotatedClass : annotatedClasses) {
			registerBean(annotatedClass);
		}
}

public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		//解析beanClass的@Conditional注解，如果有注解返回false 容器就停止处理
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}
		······
}
```

-   再看代码  这是整个容器的关键，调用了registerAnnotationConfigProcessors方法  该方法的作用
    注册和解析spring注解相关的post processors
    该方法被调用有两个地方
	 *  1.在spring使用AnnotationConfigBeanDefinitionParser解析xml文件的时候  也就是配置annotation-config的时候
	 *  2.启动在AnnotationConfigApplicationContext容器的时候
	 
```
AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
```

AnnotationConfigUtils

```
	public static final String AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME =
			"org.springframework.context.annotation.internalAutowiredAnnotationProcessor";

public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
		registerAnnotationConfigProcessors(registry, null);
	}

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {
		
		······
		//注册ConfigurationClassPostProcessor
        if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			//1.在spring使用AnnotationConfigBeanDefinitionParser解析xml文件的时候  也就是配置annotation-config的时候
			//2.启动在AnnotationConfigApplicationContext容器的时候
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
		
        //自动注入 注册AutowiredAnnotationBeanPostProcessor
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			//org.springframework.context.annotation.internalAutowiredAnnotationProcessor
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
		······

		return beanDefs;
	}
```

-   我们再看beanfactory注册了ConfigurationClassPostProcessor类，我们看看这个类的类图以及作用
    *   我们可以看到ConfigurationClassPostProcessor实现了BeanDefinitionRegistryPostProcessor接口,该接口在
    AbstractApplicationContext的refresh()方法中会被nvokeBeanFactoryPostProcessors(beanFactory)方法调用,就是在容器激活的时候被调用
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-15031527785463.jpg)


-   在看看ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry方法,由代码可以知道  这个类解析了spring使用@Configuration配置方式的bean

```
	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		RootBeanDefinition iabpp = new RootBeanDefinition(ImportAwareBeanPostProcessor.class);
		iabpp.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(IMPORT_AWARE_PROCESSOR_BEAN_NAME, iabpp);
		//注册了ConfigurationBeanPostProcessor
		RootBeanDefinition ecbpp = new RootBeanDefinition(EnhancedConfigurationBeanPostProcessor.class);
		ecbpp.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(ENHANCED_CONFIGURATION_PROCESSOR_BEAN_NAME, ecbpp);

		int registryId = System.identityHashCode(registry);
		if (this.registriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
		}
		if (this.factoriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + registry);
		}
		this.registriesPostProcessed.add(registryId);
		//处理@Configuration的bean
		processConfigBeanDefinitions(registry);
	}
```


-   在看看@SpringBootApplication的组合注解 就知道bean是怎么被加载的了

    ![](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-15031538735243.jpg)


    ![](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-15031539834791.jpg)

### ClassPathBeanDefinitionScanner Bean解析器的分析

-   ClassPathBeanDefinitionScanner提供了包扫描路径的方式启动容器，中间的解析方式和使用类文件的配置是一样的

```
ApplicationContext context=new AnnotationConfigApplicationContext("com.mitix.spring.context.expb");
```

```
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
			//寻找候选的BeanDefinition
			//注：注解在匹配的时候会递归的   会找这个注解的父注解
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				//scope注解设置beandefinition的scope属性
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				//如果满足AbstractBeanDefinition类型，就设置名字，就是上面获取到的那个
				if (candidate instanceof AbstractBeanDefinition) {
					/*
					 * 1.设置了beandefinition的默认需要的那些属性
					 * 
					 */
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				//如果是实现注解的beandefinition,然后把一堆注解解析成属性放进去，什么依赖啊什么的
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//检查注册的候选bean
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					//这里可能创建起来的是一个代理
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					//把代理注册到registory上，就是注册到工厂里
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

### 总结

AnnotationConfigApplicationContext类提供了另外一种启动spring容器的方式，而不是使用传统的xml文件进行配置  

具体详细的启动过程，可以查看spring的源码相关的问题


    

