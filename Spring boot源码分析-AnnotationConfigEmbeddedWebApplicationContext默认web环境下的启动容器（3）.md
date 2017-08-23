# Spring boot源码分析-AnnotationConfigEmbeddedWebApplicationContext默认web环境下的启动容器（3）

首先我们看容器的类图，相比于
![](http://osgqa4bwf.bkt.clouddn.com/2017-08-20-15032081720049.jpg)
看看这个图和AnnotationConfigApplicationContext有什么不一样的地方
1.GenericWebApplicationContext继承了GenericApplicationContext实现了ConfigurableWebApplicationContext接口  我们看一下这个接口，主要提供了web容器也就是servlet容器的相关操作

![](http://osgqa4bwf.bkt.clouddn.com/2017-08-20-15032085012832.jpg)

2.再看AnnotationConfigEmbeddedWebApplicationContext  容器重写了
AbstractApplicationContext的模板方法,在同期中提供了WebApplicationContextServletContextAwareProcessor的处理器，提供了容器注入的功能

```
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```

```
public class AnnotationConfigEmbeddedWebApplicationContext
		extends EmbeddedWebApplicationContext {

	private final AnnotatedBeanDefinitionReader reader;

	private final ClassPathBeanDefinitionScanner scanner;

	private Class<?>[] annotatedClasses;

	private String[] basePackages;
	
	······
	
	@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		super.postProcessBeanFactory(beanFactory);
		if (this.basePackages != null && this.basePackages.length > 0) {
			this.scanner.scan(this.basePackages);
		}
		if (this.annotatedClasses != null && this.annotatedClasses.length > 0) {
			this.reader.register(this.annotatedClasses);
		}
	}
	
	}
```


EmbeddedWebApplicationContext

```
public class EmbeddedWebApplicationContext extends GenericWebApplicationContext {

······

@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(
				new WebApplicationContextServletContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
	}
}
```


-   其他的功能基本和AnnotationConfigApplicationContext类似  可以去看上一章的解释

