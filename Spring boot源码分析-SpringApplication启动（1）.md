# Spring boot源码分析-SpringApplication启动（1）

SpringApplication这个类是引导和启动spring application程序的入库类，提供了静态的启动方法SpringApplication.run

我们先写一个hello world的示例代码（这里快速入门环境搭建就先不说了，如果不知道可以百度）
   ![1503109393144](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-1503109393144.jpg)
  
   
HelloController

```
package com.leone.chapter1.web;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }

}
```

Chapter1Application

```
package com.leone.chapter1;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Chapter1Application {

	public static void main(String[] args) {
		SpringApplication.run(Chapter1Application.class, args);
	}

}
```
启动就可以提供hello world的Rest服务

### 1.首先看SpringApplication对象的创建

* 我们可以看到新建了SpringApplication对象，并调用了run方法返回ConfigurableApplicationContext  可配置的容器接口
```
	public static ConfigurableApplicationContext run(Object source, String... args) {
		return run(new Object[] { source }, args);
	}
	
	public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
	}
```

* 再看SpringApplication的构造方法 提供了一个初始化的方法(sources传入的为Chapter1Application类)

```
	public SpringApplication(Object... sources) {
		initialize(sources);
	}
```

* 我们可以看到做了下面几件事情
    
1.      记录是否是web环境
2.      设置ApplicationContextInitializer接口的bean
3.      设置ApplicationListener接口的bean

```
	private void initialize(Object[] sources) {
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
		//1.记录是否是web环境
		this.webEnvironment = deduceWebEnvironment();
		//2.设置ApplicationContextInitializer接口的bean
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		//3.设置ApplicationListener接口的bean
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		//4.记录了main函数
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

* 先看第一点 记录是否是web环境  判断的依据是是否存在 Servlet 或者 ConfigurableWebApplicationContext（使用Calss.forName进行试探）

```
	private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

	private boolean deduceWebEnvironment() {
		for (String className : WEB_ENVIRONMENT_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return false;
			}
		}
		return true;
	}
	
	public static boolean isPresent(String className, ClassLoader classLoader) {
		try {
			forName(className, classLoader);
			return true;
		}
		catch (Throwable ex) {
			// Class or one of its dependencies is not present...
			return false;
		}
	}
```

* 设置ApplicationContextInitializer接口进行设置  一个典型的责任链模式 手续在run模式中会进行调用，先看一下ApplicationContextInitializer接口（带接口提供了一个回调方法，在ApplicationContext容器在执行refresh()方法以前进行回调）

**注：refresh() 方法是Spring的ApplicationContext启动的最重要的模板方法，如果有疑问可以查询spring相关资料**

```
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);

}
```

* 再回到第二点 ，我们再看怎么设置ApplicationContextInitializer
    
    SpringApplication提供了一个Lis用于存储接口对象
    通过扫描环境中的 META-INF/spring.factories 配置文件找到ApplicationContextInitializer相应的实现类，并进行加载存入List，提供给后面的方法使用
    
    
```
private List<ApplicationContextInitializer<?>> initializers;

public void setInitializers(
			Collection<? extends ApplicationContextInitializer<?>> initializers) {
		this.initializers = new ArrayList<ApplicationContextInitializer<?>>();
		this.initializers.addAll(initializers);
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
}

```

* 再回到第三点 ，我们再看怎么设置ApplicationListener

    SpringApplication也提供了一个Lis用于存储接口对象，如果通过同样的方式进行扫描和存储ApplicationListener，提供给后续代码使用
    

```
private List<ApplicationListener<?>> listeners;

public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
		this.listeners = new ArrayList<ApplicationListener<?>>();
		this.listeners.addAll(listeners);
	}
```

* 再回到第四点 ，记录了main函数的   到目前为止SpringApplication对象已经创建完成

### 2.然后我们再看run方法

```
	public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
	}
```

```
	public ConfigurableApplicationContext run(String... args) {
		//1.启动监听  （可以记录启动id,taskName,启动时间等）
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		//2.设置打开 java.awt.headless   java的headless模式
		configureHeadlessProperty();
		//3.设置SpringApplicationRunListeners  run事件监听
		//他有一个默认的实现类EventPublishingRunListener  在spring-boot启动的时候  对外广播事件
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.started();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			//4.创建容器的环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
			//5.启动ApplicationContext
			context = createApplicationContext();
			//6.创建故障分析器
			analyzers = new FailureAnalyzers(context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			//7.调用context的refresh方法  初始化ApplicationContext容器
			refreshContext(context);
			//8.容器启动以后的回调方法
			afterRefresh(context, applicationArguments);
			//9.发出容器启动成功的事件
			listeners.finished(context, null);
			//10.停止启动监听器
			stopWatch.stop();
			if (this.logStartupInfo) {
				//生成启动日志  使用初始类
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```
* 代码中完成了很多事情  我们一个个来看

1.  启动监听StopWatch  该监听可以记录容器启动的事件 ID 任务名称等信息
2. 设置打开  java的headless模式
3. 通过环境中的 META-INF/spring.factories 找到所有的SpringApplicationRunListener 然后设置放入SpringApplicationRunListeners
4. 创建容器的环境
5. 创建ApplicationContext容器  就是Spring上下文容器
6. 创建故障分析器
7. 调用context的refresh方法  初始化ApplicationContext容器
8. 容器启动以后的回调方法
9. 发出容器启动成功的事件
10. 停止启动监听器

* 创建StopWatch监听器  主要是为了启动SpringBoot做一些监听  这个不是重点不着重描述了
* java的headless模式  （可以百度headless的介绍[headless介绍](https://www.oschina.net/translate/using-headless-mode-in-java-se)）

* 关于SpringApplicationRunListener  我们先看代码

```
	private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
				SpringApplicationRunListener.class, types, this, args));
	}
	
    private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```
可以看出SpringApplicationRunListeners是SpringApplicationRunListener的集合  他负责注册所有的SpringApplicationRunListener  然后统一的进行调用

SpringApplicationRunListener在Springboot环境中有一个默认的实现EventPublishingRunListener配置在spring-boot代码包中的 META-INF/spring.factories  我们可以看到EventPublishingRunListener的主要功能就是向所有的ApplicationListener 发送启动的通知  这在我们的启动代码可以看到   启动过程中会调用listeners的各个方法进行记录

```
package org.springframework.boot.context.event;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringApplicationRunListener;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ApplicationListener;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.event.ApplicationEventMulticaster;
import org.springframework.context.event.SimpleApplicationEventMulticaster;
import org.springframework.core.Ordered;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.util.ErrorHandler;

/**
 * {@link SpringApplicationRunListener} to publish {@link SpringApplicationEvent}s.
 * <p>
 * Uses an internal {@link ApplicationEventMulticaster} for the events that are fired
 * before the context is actually refreshed.
 *
 * @author Phillip Webb
 * @author Stephane Nicoll
 */
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

	private final SpringApplication application;

	private final String[] args;

	private final SimpleApplicationEventMulticaster initialMulticaster;

	public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}

	@Override
	public int getOrder() {
		return 0;
	}

	@Override
	public void started() {
		this.initialMulticaster
				.multicastEvent(new ApplicationStartedEvent(this.application, this.args));
	}

	@Override
	public void environmentPrepared(ConfigurableEnvironment environment) {
		this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
				this.application, this.args, environment));
	}

	@Override
	public void contextPrepared(ConfigurableApplicationContext context) {

	}

	@Override
	public void contextLoaded(ConfigurableApplicationContext context) {
		for (ApplicationListener<?> listener : this.application.getListeners()) {
			if (listener instanceof ApplicationContextAware) {
				((ApplicationContextAware) listener).setApplicationContext(context);
			}
			context.addApplicationListener(listener);
		}
		this.initialMulticaster.multicastEvent(
				new ApplicationPreparedEvent(this.application, this.args, context));
	}

	@Override
	public void finished(ConfigurableApplicationContext context, Throwable exception) {
		SpringApplicationEvent event = getFinishedEvent(context, exception);
		if (context != null) {
			// Listeners have been registered to the application context so we should
			// use it at this point if we can
			context.publishEvent(event);
		}
		else {
			if (event instanceof ApplicationFailedEvent) {
				this.initialMulticaster.setErrorHandler(new LoggingErrorHandler());
			}
			this.initialMulticaster.multicastEvent(event);
		}
	}

	private SpringApplicationEvent getFinishedEvent(
			ConfigurableApplicationContext context, Throwable exception) {
		if (exception != null) {
			return new ApplicationFailedEvent(this.application, this.args, context,
					exception);
		}
		return new ApplicationReadyEvent(this.application, this.args, context);
	}

	private static class LoggingErrorHandler implements ErrorHandler {

		private static Log logger = LogFactory.getLog(EventPublishingRunListener.class);

		@Override
		public void handleError(Throwable throwable) {
			logger.warn("Error calling ApplicationEventListener", throwable);
		}

	}

}
```

*注：SpringFactoriesLoader有一个重要的功能   可以在Spring容器启动前扫描新建部分的类,这些类是不受spring容器生命周期管理的*


我们可以试着自己新建ApplicationListener去接收通知
新建类

```
package com.leone.chapter1;

import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;

public class MyApplicationListener implements ApplicationListener {
	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		System.out.println("我接收到了信息！！！");
		System.out.println(event.getClass().getName());
	}
}
```

在代码的META-INF/spring.factories下进行配置
```
org.springframework.context.ApplicationListener=\
com.leone.chapter1.MyApplicationListener
```
如图所示
![1503127458471](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-1503127458471.jpg)
启动代码可以看到,springboot在启动各个阶段发送了不通的Event事件通知
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-15031274873308.jpg)
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-15031275403466.jpg)



* 我们返回接着看代码 关于容器环境的创建，根据webEnvironment判断是否为web项目创建了容器环境

```
	public ConfigurableApplicationContext run(String... args) {
		······
		try {
			······
			//4.创建容器的环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
			······
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

```
	private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		//创建默认的ConfigurableEnvironment  根据是否是web环境创建
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		listeners.environmentPrepared(environment);
		if (isWebEnvironment(environment) && !this.webEnvironment) {
			environment = convertToStandardEnvironment(environment);
		}
		return environment;
	}
```

* ApplicationContext容器的创建 也是根据是否是web环境自动创建不通的容器环境  具体创建的环境有什么能力  我们后续再进行介绍 现在只需要知道创建的事这两个容器就可以了

```
	public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
			+ "annotation.AnnotationConfigApplicationContext";

	public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework."
			+ "boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";

	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				contextClass = Class.forName(this.webEnvironment
						? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
	}
```
    
* 创建故障分析器FailureAnalyzers，处理在Spring-boot启动的时候出现的异常

```
	public FailureAnalyzers(ConfigurableApplicationContext context) {
		this(context, null);
	}

	FailureAnalyzers(ConfigurableApplicationContext context, ClassLoader classLoader) {
		Assert.notNull(context, "Context must not be null");
		this.classLoader = (classLoader == null ? context.getClassLoader() : classLoader);
		this.analyzers = loadFailureAnalyzers(this.classLoader);
		prepareFailureAnalyzers(this.analyzers, context);
	}
```

* 调用context的refresh方法  初始化ApplicationContext容器，该方法调用了Spring容器的refresh 方法  如果这一块有问题的话可以先去学习spring源码分析（这个地方还会涉及到后续springboot相关注解的解析，我们在后续继续分析）

```
	private void refreshContext(ConfigurableApplicationContext context) {
	   //刷新方法
		refresh(context);
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}
	
    protected void refresh(ApplicationContext applicationContext) {
		Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
		((AbstractApplicationContext) applicationContext).refresh();
	}
```


* 容器启动以后的回调方法，主要做一些扩展的预留
* 发出容器启动成功的事件  上面已经描述了SpringApplicationRunListeners在各个阶段都会发出通知  这个就是完成以后的地方
* 启动完成  停止监听器  然后生成相关的启动日志
* 当然还有如果启动异常会进行异常处理


```
public ConfigurableApplicationContext run(String... args) {
		······
		try {
        ······
			//容器启动以后的回调方法
			afterRefresh(context, applicationArguments);
			//发出容器启动成功的事件
			listeners.finished(context, null);
			//停止启动监听器
			stopWatch.stop();
			if (this.logStartupInfo) {
				//生成启动日志  使用初始类
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
		  //启动异常的处理
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```


到此为止  SpringApplication对于spring boot的静态启动工作已经介绍完毕

