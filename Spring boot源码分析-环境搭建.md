# Spring boot源码分析-环境搭建

### 源码的下载 

*   springboot源码托管在github [spring-boot](https://github.com/spring-projects/spring-boot)
![1503053079223](http://osgqa4bwf.bkt.clouddn.com/2017-08-18-1503053079223.jpg)

*   Fork spring-boot源码（fork完成以后可以自行修改源码）
![1503053079224](http://osgqa4bwf.bkt.clouddn.com/2017-08-18-1503053079224.jpeg)

*   克隆代码到本地仓库
![屏幕快照 2017-08-18 下午11.17.13](http://osgqa4bwf.bkt.clouddn.com/2017-08-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-18%20%E4%B8%8B%E5%8D%8811.17.13.png)


### 源码构建

* 查看源码地址  找到 Building from Source [CONTRIBUTING.adoc](https://github.com/infoconsole/spring-boot/blob/master/CONTRIBUTING.adoc)

* 看到Working with the code上面关于 Building from source的介绍  大致的意思：

    >  -  建议使用 Spring Tools Suite or Eclipse 来构建代码   不过个人建议使用idea
    
    >  - 使用maven 3.2.1或者更高的版本  使用jdk1.8
    
    >  - 默认的构建方式使用maven命令 
        *$ ./mvnw clean install*
        
        提示：可以设置maven的环境 MAVEN_OPTS  -Xmx512m
        
    >  - 如果你是重新构建的  可以直接使用下列命令跳过检查  
        *$ ./mvnw clean install -DskipTests -Pfast*
        
    >  - 通过两阶段进行全量构建
        
    >  1）Prepare the build  准备构建 安装spring-boot-maven-plugin插件 
         *$ ./mvnw -P snapshot,prepare install -DskipTests*
         
    >  2) Run the full build 执行构建任务
        *$ ./mvnw -s ./settings.xml -f spring-boot-full-build -P full clean install*
        
         
* 下面我们看一下全量构建
    - 执行 *$ ./mvnw -P snapshot,prepare install -DskipTests*
    ![屏幕快照 2017-08-18 下午11.51.53](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-18%20%E4%B8%8B%E5%8D%8811.51.53.png)
![屏幕快照 2017-08-18 下午11.52.18](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-18%20%E4%B8%8B%E5%8D%8811.52.18.png)

- 执行（执行时间会比较久）  *$ ./mvnw -s ./settings.xml -f spring-boot-full-build -P full clean install*
![屏幕快照 2017-08-18 下午11.55.37](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-18%20%E4%B8%8B%E5%8D%8811.55.37.png)
![屏幕快照 2017-08-18 下午11.55.59](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-18%20%E4%B8%8B%E5%8D%8811.55.59.png)

- 执行成功以后 可以通过idea导入maven工程 
![屏幕快照 2017-08-19 上午12.04.14](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-19%20%E4%B8%8A%E5%8D%8812.04.14.png)
选择通过已经存在的代码新建
![屏幕快照 2017-08-19 上午12.21.17](http://osgqa4bwf.bkt.clouddn.com/2017-08-19-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-08-19%20%E4%B8%8A%E5%8D%8812.21.17.png)
选择maven工程，然后默认完成余下的向导操作

* 接下来，解决一些maven依赖的问题  就可以进行学习之旅了

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::  


