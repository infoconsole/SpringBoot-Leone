## Spring boot源码分析-Conditional（12）
*   Condition是spring4.0增加的条件判断接口，用于判断条件满足情况，目前在spring中使用Conditional有两个地方

    1.  在注册bean的时候会拿Condition判断是否这个是一个满足条件应该注册的bean
    
AnnotatedBeanDefinitionReader.registerBean    

```
    /*
	 * 这个方法其实是把自己给注册上了（在AnnotationConfigApplicationContext启动的时候）
	 */
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

ClassPathScanningCandidateComponentProvider.findCandidateComponents

```
/**
	 * Scan the class path for candidate components.
	 * @param basePackage the package to check for annotated classes
	 * @return a corresponding Set of autodetected bean definitions
	 * 扫描得到候选组件
	 */
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		......
						//满足是一个组件的所有要求
						if (isCandidateComponent(metadataReader)) {
							//生成ScannedGenericBeanDefinition其实就是一个带有注解信息，没有任何构造函数参数等等的GenericBeanDefinition
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							sbd.setResource(resource);
							sbd.setSource(resource);
							//判断是否给定的beandefinition是一个具体的类，不是接口或者抽象类
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						......
		return candidates;
	}
```

```
	private boolean isConditionMatch(MetadataReader metadataReader) {
		if (this.conditionEvaluator == null) {
			this.conditionEvaluator = new ConditionEvaluator(getRegistry(), getEnvironment(), getResourceLoader());
		}
		return !this.conditionEvaluator.shouldSkip(metadataReader.getAnnotationMetadata());
	}
```
    
    2.  在Spring按照Configuration类的方式进行配置的时候，会加载解析Configuration组件类，加载的时候会调用Condition的子类ConfigurationCondition去判断是否需要解析Configuration或者是否注册Bean（查看ConfigurationClassPostProcessor）
    
*   ConfigurationCondition增加了解释的方式判断，Condition判断的扩展，在解析Configuration的时候使用


```
public interface ConfigurationCondition extends Condition {
    
	ConfigurationPhase getConfigurationPhase();

	public static enum ConfigurationPhase {

		PARSE_CONFIGURATION,

		REGISTER_BEAN
	}

}
```
    
*   SpringBootCondition实现了Condition的抽象类，提供动态类加载功能，提供了日志和诊断功能


```
public abstract class SpringBootCondition implements Condition {

	private final Log logger = LogFactory.getLog(getClass());

	@Override
	public final boolean matches(ConditionContext context,
			AnnotatedTypeMetadata metadata) {
		String classOrMethodName = getClassOrMethodName(metadata);
		try {
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			logOutcome(classOrMethodName, outcome);
			recordEvaluation(context, classOrMethodName, outcome);
			return outcome.isMatch();
		}
		catch (NoClassDefFoundError ex) {
			throw new IllegalStateException(
					"Could not evaluate condition on " + classOrMethodName + " due to "
							+ ex.getMessage() + " not "
							+ "found. Make sure your own configuration does not rely on "
							+ "that class. This can also happen if you are "
							+ "@ComponentScanning a springframework package (e.g. if you "
							+ "put a @ComponentScan in the default package by mistake)",
					ex);
		}
		catch (RuntimeException ex) {
			throw new IllegalStateException(
					"Error processing condition on " + getName(metadata), ex);
		}
	}

	private String getName(AnnotatedTypeMetadata metadata) {
		if (metadata instanceof AnnotationMetadata) {
			return ((AnnotationMetadata) metadata).getClassName();
		}
		if (metadata instanceof MethodMetadata) {
			MethodMetadata methodMetadata = (MethodMetadata) metadata;
			return methodMetadata.getDeclaringClassName() + "."
					+ methodMetadata.getMethodName();
		}
		return metadata.toString();
	}

	private static String getClassOrMethodName(AnnotatedTypeMetadata metadata) {
		if (metadata instanceof ClassMetadata) {
			ClassMetadata classMetadata = (ClassMetadata) metadata;
			return classMetadata.getClassName();
		}
		MethodMetadata methodMetadata = (MethodMetadata) metadata;
		return methodMetadata.getDeclaringClassName() + "#"
				+ methodMetadata.getMethodName();
	}

	private void logOutcome(String classOrMethodName, ConditionOutcome outcome) {
		if (this.logger.isTraceEnabled()) {
			this.logger.trace(getLogMessage(classOrMethodName, outcome));
		}
	}

	private StringBuilder getLogMessage(String classOrMethodName,
			ConditionOutcome outcome) {
		StringBuilder message = new StringBuilder();
		message.append("Condition ");
		message.append(ClassUtils.getShortName(getClass()));
		message.append(" on ");
		message.append(classOrMethodName);
		message.append(outcome.isMatch() ? " matched" : " did not match");
		if (StringUtils.hasLength(outcome.getMessage())) {
			message.append(" due to ");
			message.append(outcome.getMessage());
		}
		return message;
	}

	private void recordEvaluation(ConditionContext context, String classOrMethodName,
			ConditionOutcome outcome) {
		if (context.getBeanFactory() != null) {
			ConditionEvaluationReport.get(context.getBeanFactory())
					.recordConditionEvaluation(classOrMethodName, this, outcome);
		}
	}

	/**
	 * Determine the outcome of the match along with suitable log output.
	 * @param context the condition context
	 * @param metadata the annotation metadata
	 * @return the condition outcome
	 */
	public abstract ConditionOutcome getMatchOutcome(ConditionContext context,
			AnnotatedTypeMetadata metadata);

	/**
	 * Return true if any of the specified conditions match.
	 * @param context the context
	 * @param metadata the annotation meta-data
	 * @param conditions conditions to test
	 * @return {@code true} if any condition matches.
	 */
	protected final boolean anyMatches(ConditionContext context,
			AnnotatedTypeMetadata metadata, Condition... conditions) {
		for (Condition condition : conditions) {
			if (matches(context, metadata, condition)) {
				return true;
			}
		}
		return false;
	}

	/**
	 * Return true if any of the specified condition matches.
	 * @param context the context
	 * @param metadata the annotation meta-data
	 * @param condition condition to test
	 * @return {@code true} if the condition matches.
	 */
	protected final boolean matches(ConditionContext context,
			AnnotatedTypeMetadata metadata, Condition condition) {
		if (condition instanceof SpringBootCondition) {
			return ((SpringBootCondition) condition).getMatchOutcome(context, metadata)
					.isMatch();
		}
		return condition.matches(context, metadata);
	}

}
```
    



