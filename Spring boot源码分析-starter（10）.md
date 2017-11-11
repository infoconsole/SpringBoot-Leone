# Spring boot源码分析-starter（10）
springboot的starter作为开箱即用的启动方式，给开发带来了巨大的便利程度。现在我们就来学习一下starter.

### springboot是靠什么实现starter
我们可以观察一下spring-boot-starters包，存放着springboot自带的常用的starter功能列表
![](http://osgqa4bwf.bkt.clouddn.com/2017-11-11-15085722251084.jpg)
![](http://osgqa4bwf.bkt.clouddn.com/2017-11-11-15085750447424.jpg)

### starter核心spring-boot-starter工程
spring-boot-starter为空工程，主要的作用是做了包依赖管理
![](http://osgqa4bwf.bkt.clouddn.com/2017-11-11-15085754193877.jpg)
从POM.XML文件我们可以看出，他引用了spring-boot-autoconfigure，也就加载了springboot封装的常用的开箱即用功能，只要运行环境中存在jar包，功能就会启动，他是所有其他的starter的父工程（除了logging）

### 自己编写一个starter工程
*   创建一个configure工程
*   创建starter工程
*   需要的可以查看springboot默认的starter编写（有好的项目再编写）





