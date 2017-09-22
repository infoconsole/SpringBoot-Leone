# Spring boot源码分析-log日志系统（6）

说到日志系统的启动，我们首先看LoggingApplicationListener，这个类就是springboot日志系统加载的入口,可以看出实现了ApplicationListener，在上一节我们分析过ApplicationListener的运行方式
![](http://osgqa4bwf.bkt.clouddn.com/2017-09-22-15059570751290.jpg)

*   先看onApplicationEvent方法，从方法中我们可以看到，在springboot在几个阶段都会调用onApplicationEvent方法

```
		@Override
	public void onApplicationEvent(ApplicationEvent event) {
		//在springboot启动的时候
		if (event instanceof ApplicationStartedEvent) {
			onApplicationStartedEvent((ApplicationStartedEvent) event);
		}
		//springboot的Environment环境准备完成的时候
		else if (event instanceof ApplicationEnvironmentPreparedEvent) {
			onApplicationEnvironmentPreparedEvent(
					(ApplicationEnvironmentPreparedEvent) event);
		}
		//在springboot容器的环境设置完成以后
		else if (event instanceof ApplicationPreparedEvent) {
			onApplicationPreparedEvent((ApplicationPreparedEvent) event);
		}
		//容器关闭的时候
		else if (event instanceof ContextClosedEvent && ((ContextClosedEvent) event)
				.getApplicationContext().getParent() == null) {
			onContextClosedEvent();
		}
		//容器启动失败的时候
		else if (event instanceof ApplicationFailedEvent) {
			onApplicationFailedEvent();
		}
	}
```    
* onApplicationStartedEvent方法，在springboot开始启动时调用，主要工作为获取LoggingSystem然后调用beforeInitialize进行初始化LoggingSystem前的设置

```
	private void onApplicationStartedEvent(ApplicationStartedEvent event) {
		this.loggingSystem = LoggingSystem
				.get(event.getSpringApplication().getClassLoader());
		this.loggingSystem.beforeInitialize();
	}
```
首先设置LoggingSystem，当运行参数配置
-Dorg.springframework.boot.logging.LoggingSystem的时候，会根据配置的参数加载
默认支持的LoggingSystem有3个，当没有配置时默认加载LogbackLoggingSystem

```
	static {
		Map<String, String> systems = new LinkedHashMap<String, String>();
		systems.put("ch.qos.logback.core.Appender",
				"org.springframework.boot.logging.logback.LogbackLoggingSystem");
		systems.put("org.apache.logging.log4j.core.impl.Log4jContextFactory",
				"org.springframework.boot.logging.log4j2.Log4J2LoggingSystem");
		systems.put("java.util.logging.LogManager",
				"org.springframework.boot.logging.java.JavaLoggingSystem");
		SYSTEMS = Collections.unmodifiableMap(systems);
	}
```

* onApplicationEnvironmentPreparedEvent方法，在springboot完成环境初始化以后进行调用
* 
```
	private void onApplicationEnvironmentPreparedEvent(
			ApplicationEnvironmentPreparedEvent event) {
		if (this.loggingSystem == null) {
			this.loggingSystem = LoggingSystem
					.get(event.getSpringApplication().getClassLoader());
		}
		initialize(event.getEnvironment(), event.getSpringApplication().getClassLoader());
	}
```

主要是设置相关的参数，进行初始化，

```
	protected void initialize(ConfigurableEnvironment environment,
			ClassLoader classLoader) {
		//设置相关的环境参数SystemProperty
		new LoggingSystemProperties(environment).apply();
		LogFile logFile = LogFile.get(environment);
		if (logFile != null) {
			logFile.applyToSystemProperties();
		}
		//environment参数debug   trace  设置日志级别
		initializeEarlyLoggingLevel(environment);
		//environment参数logging.config，初始化loggingSystem
		initializeSystem(environment, this.loggingSystem, logFile);
		//设置springboot默认的一些日志级别
		initializeFinalLoggingLevels(environment, this.loggingSystem);
		//注册ShutdownHook
		registerShutdownHookIfNecessary(environment, this.loggingSystem);
	}
```
>    new LoggingSystemProperties(environment).apply();的主要作用是设置logging.
> exception-conversion-word
> pattern.console
> pattern.file
> pattern.level到System环境中


```
	public void apply(LogFile logFile) {
		RelaxedPropertyResolver propertyResolver = RelaxedPropertyResolver
				.ignoringUnresolvableNestedPlaceholders(this.environment, "logging.");
		setSystemProperty(propertyResolver, EXCEPTION_CONVERSION_WORD,
				"exception-conversion-word");
		setSystemProperty(propertyResolver, CONSOLE_LOG_PATTERN, "pattern.console");
		setSystemProperty(propertyResolver, FILE_LOG_PATTERN, "pattern.file");
		setSystemProperty(propertyResolver, LOG_LEVEL_PATTERN, "pattern.level");
		setSystemProperty(PID_KEY, new ApplicationPid().toString());
		if (logFile != null) {
			logFile.applyToSystemProperties();
		}
	}
```

>    initializeEarlyLoggingLevel(environment);的主要作用是根据environment环境中的debug或者trace属性设置日志级别springBootLogging（后面代码会使用到）

```
	private void initializeEarlyLoggingLevel(ConfigurableEnvironment environment) {
		if (this.parseArgs && this.springBootLogging == null) {
			if (isSet(environment, "debug")) {
				this.springBootLogging = LogLevel.DEBUG;
			}
			if (isSet(environment, "trace")) {
				this.springBootLogging = LogLevel.TRACE;
			}
		}
	}
```
>    initializeFinalLoggingLevels(environment, this.loggingSystem);做了两件事
> 1设置springboot内置的log日志级别  debug或者trace
> 通过logging.level.* 设置第三方的包的日志

```
	static {
		LOG_LEVEL_LOGGERS = new LinkedMultiValueMap<LogLevel, String>();
		LOG_LEVEL_LOGGERS.add(LogLevel.DEBUG, "org.springframework.boot");
		LOG_LEVEL_LOGGERS.add(LogLevel.TRACE, "org.springframework");
		LOG_LEVEL_LOGGERS.add(LogLevel.TRACE, "org.apache.tomcat");
		LOG_LEVEL_LOGGERS.add(LogLevel.TRACE, "org.apache.catalina");
		LOG_LEVEL_LOGGERS.add(LogLevel.TRACE, "org.eclipse.jetty");
		LOG_LEVEL_LOGGERS.add(LogLevel.TRACE, "org.hibernate.tool.hbm2ddl");
        LOG_LEVEL_LOGGERS.add(LogLevel.DEBUG, "org.hibernate.SQL");
	}
```

