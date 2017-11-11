## Spring boot源码分析-ConfigurationProperties（11）
* 该功能可以实现批量添加Properties参数

```
@ConfigurationProperties(locations = "classpath:test.properties",
		prefix = "leone")
public class TestProperties {
	private String name;
	private int age;
	private Mark mark;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public Mark getMark() {
		return mark;
	}

	public void setMark(Mark mark) {
		this.mark = mark;
	}
}
```

```
@SpringBootApplication
@EnableConfigurationProperties(TestProperties.class)
public class Chapter2Application {

	public static void main(String[] args) {
		SpringApplication.run(Chapter2Application.class, args);
	}

}
```

test.properties

```
leone.name=infotech
leone.age=99
leone.mark.dmark=dddddd
```
* 工作原理
前面我们已经分析过了，spring的starter再启动的时候回加载configure

那么我们去看看spring-boot-autoconfigure的配置文件spring.factories
可以看到org.springframework.boot.autoconfigure.EnableAutoConfiguration的配置中有一个org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration继续看进去，calss上有个注解EnableConfigurationProperties，再进去发现是一个EnableConfigurationPropertiesImportSelector  
这时候我们就知道了，在启动的过程中  spring会去EnableConfigurationPropertiesImportSelector来注册bean  并且处理


