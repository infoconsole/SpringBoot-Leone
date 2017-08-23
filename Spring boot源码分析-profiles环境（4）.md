# Spring boot源码分析-profiles环境（4）

###     spring中profiles的环境应用

我们先看一下spring环境中profiles的使用

![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15033917737289.jpg)

MyTestBean

```
package com.mitix;

import org.springframework.beans.factory.annotation.Value;

/**
 * @version 1.0.0
 * @author oldflame-Jm first example this is a pojo
 */
public class MyTestBean {

	@Value("${test.teststr}")
	private String testStr = "";

	public String getTestStr() {
		return testStr;
	}

	public void setTestStr(String testStr) {
		this.testStr = testStr;
	}
}
```

applicationContext.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
						http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
						http://www.springframework.org/schema/context
						http://www.springframework.org/schema/context/spring-context-4.3.xsd">
    <context:annotation-config></context:annotation-config>
    <bean id="mytestbean" class="com.mitix.MyTestBean">
    </bean>
    <beans profile="dev">
        <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
            <property name="locations">
                <list>
                    <value>dev.properties</value>
                </list>
            </property>
        </bean>
    </beans>
    <!-- 定义生产使用的profile -->
    <beans profile="prod">
        <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
            <property name="locations">
                <list>
                    <value>prod.properties</value>
                </list>
            </property>
        </bean>
    </beans>
</beans>
```

dev.properties

```
test.teststr=hello infotech dev
```

prod.properties

```
test.teststr=hello infotech prod
```

-   使用方式1：使用JVM参数设置
    
```
package com.mitix;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class ApplicationContextStart {
    @SuppressWarnings("resource")
    public static void main(String[] args) {
        //测试ApplicationContext第一个Beans实例
        ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        MyTestBean bean = (MyTestBean) context.getBean("mytestbean");
        System.out.println(bean.getTestStr());
    }
}
```
配置 -Dspring.profiles.active="dev" 启动
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15033919169988.jpg)

启动以后显示
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15033919553541.jpg)


-   使用方式2：使用spring-test,可以使用类配置和xml配置两种方式


BeanConfiguration
```
package com.mitix;

import org.springframework.beans.factory.config.PropertyPlaceholderConfigurer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.core.io.ClassPathResource;

@Configuration
public class BeanConfiguration {

    @Bean(name = "mytestbean")
    public MyTestBean myTestBean() {
        return new MyTestBean();
    }

    @Bean
    @Profile("dev")
    public PropertyPlaceholderConfigurer devPropertyPlaceholderConfigurer() {
        PropertyPlaceholderConfigurer propertyPlaceholderConfigurer=new PropertyPlaceholderConfigurer();
        propertyPlaceholderConfigurer.setLocation(new ClassPathResource("com/mitix/dev.properties"));
        return propertyPlaceholderConfigurer;
    }

    @Bean
    @Profile("prod")
    public PropertyPlaceholderConfigurer prodPropertyPlaceholderConfigurer() {
        PropertyPlaceholderConfigurer propertyPlaceholderConfigurer=new PropertyPlaceholderConfigurer();
        propertyPlaceholderConfigurer.setLocation(new ClassPathResource("com/mitix/prod.properties"));
        return propertyPlaceholderConfigurer;
    }
}

```
ApplicationContextTest

```
package com.mitix;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.AbstractJUnit4SpringContextTests;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
//@ContextConfiguration(classes = {BeanConfiguration.class})
@ContextConfiguration(locations = {"classpath:applicationContext.xml"})
@ActiveProfiles("prod")
public class ApplicationContextTest extends AbstractJUnit4SpringContextTests {

    @Test
    public void profilesTest() {
        MyTestBean myTestBean= (MyTestBean) applicationContext.getBean("mytestbean");
        System.out.println(myTestBean.getTestStr());
    }
}
```

运行结果显示
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15033921337975.jpg)


### spring中profiles加载分析

-   从上面的示例我们可以看到profiles的功能，我们再去看加载的源码，首先profiles的数据是存储在spring的environment 中  看一下类的结构
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15033925700305.jpg)

其中AbstractEnvironment 提供了activeProfiles用于存放激活的profiles 提供了defaultProfiles作为默认的profiles,以及获取，设置和增加profiles的方法

```
public abstract class AbstractEnvironment implements ConfigurableEnvironment {
	public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";
	public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";

	private final Set<String> activeProfiles = new LinkedHashSet<String>();

	private final Set<String> defaultProfiles = new LinkedHashSet<String>(getReservedDefaultProfiles());
		private final MutablePropertySources propertySources = new MutablePropertySources(this.logger);

	protected Set<String> doGetActiveProfiles() {
		synchronized (this.activeProfiles) {
			if (this.activeProfiles.isEmpty()) {
				String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.activeProfiles;
		}
	}
	
		@Override
	public void setActiveProfiles(String... profiles) {
		Assert.notNull(profiles, "Profile array must not be null");
		synchronized (this.activeProfiles) {
			this.activeProfiles.clear();
			for (String profile : profiles) {
				validateProfile(profile);
				this.activeProfiles.add(profile);
			}
		}
	}

	@Override
	public void addActiveProfile(String profile) {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug(String.format("Activating profile '%s'", profile));
		}
		validateProfile(profile);
		doGetActiveProfiles();
		synchronized (this.activeProfiles) {
			this.activeProfiles.add(profile);
		}
	}
}
```

-   然后我们再看一下，profiles是如何工作的，我们首先看解析配置文件
Context在解析配置文件的时候

```
	/**
	 * Register each bean definition within the given root {@code <beans/>} element.
	 */
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
						//判断profiles属性的值是否满足环境中的profiles
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
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
如果为beans的节点的时候，就会去获取当前的节点是否有profiles属性，属性值是否在生效的profiles里面，再看判断条件

```
	@Override
	public boolean acceptsProfiles(String... profiles) {
		Assert.notEmpty(profiles, "Must specify at least one profile");
		for (String profile : profiles) {
			if (StringUtils.hasLength(profile) && profile.charAt(0) == '!') {
				if (!isProfileActive(profile.substring(1))) {
					return true;
				}
			}
			else if (isProfileActive(profile)) {
				return true;
			}
		}
		return false;
	}
```

```
	protected boolean isProfileActive(String profile) {
		validateProfile(profile);
		Set<String> currentActiveProfiles = doGetActiveProfiles();
		return (currentActiveProfiles.contains(profile) ||
				(currentActiveProfiles.isEmpty() && doGetDefaultProfiles().contains(profile)));
	}
```

```
	protected Set<String> doGetActiveProfiles() {
		synchronized (this.activeProfiles) {
			if (this.activeProfiles.isEmpty()) {
				String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.activeProfiles;
		}
	}
```
我们可以看到，profiles参数是从AbstractEnvironment的propertySources中获取的。该参数的最终来源就是在Spring初始化容器，新建Environment 的时候设置的参数（我们可以看到AbstractApplicationContext的refresh()方法中，在新建StandardEnvironment标准的spring容器环境的时候  会设置systemProperties，systemEnvironment两个属性集合）

```
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();
			······
			}
```

```
protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
	
```

```
	public AbstractEnvironment() {
		customizePropertySources(this.propertySources);
		if (this.logger.isDebugEnabled()) {
			this.logger.debug(String.format(
					"Initialized %s with PropertySources %s", getClass().getSimpleName(), this.propertySources));
		}
	}
```

```
	@Override
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
		propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
	}
```
我们可以看一下启动容器新建StandardEnvironment的信息
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034501100858.jpg)

![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034501567302.jpg)
在systemProperties参数中存着所有的运行参数  其中包括spring.profiles.active参数

-   总结分析 ，在spring中，所有的profiles都是存在environment环境中的，只要保证context在新建完成以后设置生效profiles，就可以应用于整个系统。

###     springboot中profiles的环境应用

-   先看使用

![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034625668343.jpg)


HelloController
```
package com.leone.chapter.profiles.web;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

	@Value("${profiles.load.name}")
	private String name;

    @RequestMapping("/hello")
    public String index() {
		return "Hello World--" + name;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}
}
```

ChapterProfilesApplication

```
package com.leone.chapter.profiles;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ChapterProfilesApplication {

	public static void main(String[] args) {
		SpringApplication.run(ChapterProfilesApplication.class, args);
	}

}
```

application.properties
```
server.context-path=/
server.port=8080
spring.profiles.active=dev
```

application-dev.properties
```
profiles.load.name= this is dev profiles
```

application-prod.properties
```
profiles.load.name= this is prod profiles
```

当我么访问的时候，返回的是
Hello World--this is dev profiles


-   当在启动参数中配置

得到的结果是
Hello World--this is prod profiles
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034628139204.jpg)


### springboot中profiles加载分析
我么看springboot的启动代码中，run方法的代码:

```
public ConfigurableApplicationContext run(String... args) {
		······
		try {
        
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			//创建容器的环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
			//启动ApplicationContext
			context = createApplicationContext();
			//创建故障分析器
			analyzers = new FailureAnalyzers(context);
			prepareContext(context, environment, listeners, applicationArguments,
			······
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```
-   1我们先看环境的创建

```
	private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		//创建默认的ConfigurableEnvironment  根据是否是web环境创建
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		//默认的环境参数设置
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		//springboot使用SpringApplicationRunListeners通知机制 在环境创建完成进行了设置
		listeners.environmentPrepared(environment);
		if (isWebEnvironment(environment) && !this.webEnvironment) {
			environment = convertToStandardEnvironment(environment);
		}
		return environment;
	}
```
首先创建一个默认的环境，根据启动的容器是否是web容器创建StandardServletEnvironment 或者StandardEnvironment

```
	private ConfigurableEnvironment getOrCreateEnvironment() {
		if (this.environment != null) {
			return this.environment;
		}
		if (this.webEnvironment) {
			return new StandardServletEnvironment();
		}
		return new StandardEnvironment();
	}
```
查看类的模型结构
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034671395261.jpg)
可知和在spring容器中启动的事一样的，我么可以预见在启动完成以后系统参数systemProperties，systemEnvironment已经存在
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034673505380.jpg)

-   2然后进行容器的本地设置configureEnvironment


```
	protected void configureEnvironment(ConfigurableEnvironment environment,
			String[] args) {
		configurePropertySources(environment, args);
		//profiles
		configureProfiles(environment, args);
	}
```

首先进行陪PropertySources设置，如果有设置defaultProperties，那么增加defaultProperties这个source选项（一般情况没有进行设置）

然后看传入的参数，保存为commandLineArgs的PropertySources
注意：添加的位置是在第一个（优先级最高）,当运行参数program arguments中设置--spring.profiles.active="prod" 时也能生效，而且优先级别最高，但是一般不建议这个么设置

```
	protected void configurePropertySources(ConfigurableEnvironment environment,
			String[] args) {
		MutablePropertySources sources = environment.getPropertySources();
		//设置默认的defaultProperties属性
		if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
			sources.addLast(
					new MapPropertySource("defaultProperties", this.defaultProperties));
		}
		//增加参数的defaultProperties
		if (this.addCommandLineProperties && args.length > 0) {
			String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
			if (sources.contains(name)) {
				PropertySource<?> source = sources.get(name);
				CompositePropertySource composite = new CompositePropertySource(name);
				composite.addPropertySource(new SimpleCommandLinePropertySource(
						name + "-" + args.hashCode(), args));
				composite.addPropertySource(source);
				sources.replace(name, composite);
			}
			else {
				sources.addFirst(new SimpleCommandLinePropertySource(args));
			}
		}
	}
```

进行profiles的设置configureProfiles，主要是初始化设置一下setActiveProfiles（可以从系统参数中取出spring.profiles.active设置到Environment环境的activeProfiles中去），使用了
environment.setActiveProfiles  说明是重置不是增加profiles

```
	protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
		environment.getActiveProfiles(); // ensure they are initialized
		// But these ones should go first (last wins in a property key clash)
		Set<String> profiles = new LinkedHashSet<String>(this.additionalProfiles);
		profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
		environment.setActiveProfiles(profiles.toArray(new String[profiles.size()]));
	}
```
结合这个environment.getActiveProfiles方法我们可以知道，主要功能是获取了一下系统参数进行设置，当profiles已经设置过就把原来的取出来设置回去，其实没有变化

```
	@Override
	public String[] getActiveProfiles() {
		return StringUtils.toStringArray(doGetActiveProfiles());
	}

	protected Set<String> doGetActiveProfiles() {
		synchronized (this.activeProfiles) {
			if (this.activeProfiles.isEmpty()) {
				String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.activeProfiles;
		}
	}
```

-   3springboot使用SpringApplicationRunListeners通知机制 springboot默认使用了EventPublishingRunListener作为事件通知的统一通知入口 	
```
listeners.environmentPrepared(environment);
```	    

当容器环境准备通知完成以后，EventPublishingRunListener负责向所有的ApplicationListener发出environmentPrepared的通知，事件为
ApplicationEnvironmentPreparedEvent  springboot的环境准备事件


SimpleApplicationEventMulticaster

```
	public void environmentPrepared(ConfigurableEnvironment environment) {
		this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
				this.application, this.args, environment));
	}
	
		@Override
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}

	@Override
	public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(new Runnable() {
					@Override
					public void run() {
						invokeListener(listener, event);
					}
				});
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
	
	protected void invokeListener(ApplicationListener listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				listener.onApplicationEvent(event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			try {
				listener.onApplicationEvent(event);
			}
			catch (ClassCastException ex) {
				String msg = ex.getMessage();
				if (msg == null || msg.startsWith(event.getClass().getName())) {
					// Possibly a lambda-defined listener which we could not resolve the generic event type for
					Log logger = LogFactory.getLog(getClass());
					if (logger.isDebugEnabled()) {
						logger.debug("Non-matching event type for listener: " + listener, ex);
					}
				}
				else {
					throw ex;
				}
			}
		}
	}
```

然后我们查看，所有的ApplicationListener中，可以处理ApplicationEnvironmentPreparedEvent事件的Listener,
从springboot源码的启动分析中我们可以看到
springboot启动的时候已经扫描所有的Listener

SpringApplication.java

```
	private void initialize(Object[] sources) {
		······
		//设置ApplicationListener接口的bean
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		······
	}
```

可以在spring-boot的spring.factories文件中找到ConfigFileApplicationListener作为环境准备完成以后的properties加载入库，默认的加载环境为  application.properties或者application.yml

我们知道了配置文件加载的方式
具体的加载看对ConfigFileApplicationListener的分析




