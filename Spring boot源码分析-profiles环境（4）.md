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
先看使用

![](http://osgqa4bwf.bkt.clouddn.com/2017-08-23-15034579973231.jpg)

HelloController
```
package com.leone.chapter.profiles.web;

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

[1111](https://github.com/infoconsole/spring-boot.git)


