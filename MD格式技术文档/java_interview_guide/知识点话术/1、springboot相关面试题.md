### 1、什么是springboot

SpringBoot是Spring项目中的一个子工程，其实人们把Spring Boot 称为搭建程序的`脚手架`。其最主要作用就是帮我们快速的构建庞大的spring项目，并且尽可能的减少一切xml配置，做到开箱即用，迅速上手，让我们关注与业务而非配置。



### 2、为什么要用springboot

Spring Boot 优点非常多，如：
一、独立运行
Spring Boot而且内嵌了各种servlet容器，Tomcat、Jetty等，现在不再需要打成war包部署到容器中，Spring Boot只要打成一个可执行的
jar包就能独立运行，所有的依赖包都在一个jar包内。
二、简化配置
spring-boot-starter-web启动器自动依赖其他组件，简少了maven的配置。
三、自动配置
Spring Boot能根据当前类路径下的类、jar包来自动配置bean，如添加一个spring-boot-starter-web启动器就能拥有web的功能，无需其他配置。
四、无代码生成和XML配置
Spring Boot配置过程中无代码生成，也无需XML配置文件就能完成所有配置工作，这一切都是借助于条件注解完成的，这也是Spring4.x的核心功能之一。
五、应用监控
Spring Boot提供一系列端点可以监控服务及应用，做健康检测



### 3、springboot有哪些优点

1. 减少开发，测试时间和努力。
2. 使用 JavaConﬁg 有助于避免使用 XML。
3. 避免大量的 Maven 导入和各种版本冲突。
5. 通过提供默认值快速开始开发。
6. 没有单独的 Web 服务器需要。这意味着你不再需要启动 Tomcat，Glassﬁsh或其他任何东西。
7. 需要更少的配置 因为没有 web.xml 文件。只需添加用@ Conﬁguration 注释的类，然后添加用@Bean 注释的方法，Spring 将自动加载对象并像以前一样对其进行管理。您甚至可以将@Autowired 添加到 bean 方法中，以使 Spring 自动装入需要的依赖关系中。
8. 基于环境的配置 使用这些属性，您可以将您正在使用的环境传递到应用程序：-Dspring.proﬁles.active = {enviornment}。在加载主应用程序属性文件后，Spring 将在（application{environment} .properties）中加载后续的应用程序属性文件。

application.yml

​	spring.active.profile=dev

application-dev.yml

application-test.yml

application-prod.yml

 

### 4、Spring Boot 的核心注解是哪个？它主要由哪几个注解组成的？

启动类上面的注解是@SpringBootApplication，它也是 Spring Boot 的核心注解，主要组合包含了以下
3 个注解：
@SpringBootConﬁguration：组合了 @Conﬁguration 注解，实现配置文件的功能。
@EnableAutoConﬁguration：打开自动配置的功能，也可以关闭某个自动配置的选项，如关闭数据源自动配置功能：@SpringBootApplication(exclude = { DataSourceAutoConﬁguration.class })。
@ComponentScan：Spring组件扫描，从当前类所在的包以及子包扫描，之外的包扫描不到，所以我们在开发的时候，所有的类都在主类的子包下



### 5、springboot项目有哪几种运行方式

1. 打包用命令或者放到容器中运行
2. 用 Maven/Gradle 插件运行
3. 直接执行 main 方法运行



### 6、如何理解springboot中的starters？

Starters可以理解为启动器，它包含了一系列可以集成到应用里面的依赖包，你可以一站式集成Spring及其他技术，而不需要到处找示例代码和依赖包。如你想使用Spring JPA访问数据库，只要加入springboot-starter-data-jpa启动器依赖就能使用了。Starters包含了许多项目中需要用到的依赖，它们能快速持续的运行，都是一系列得到支持的管理传递性依赖。



### 7、springboot自动配置原理

这个就得从springboot项目的核心注解@SpringbootApplication说起了，这个注解包含了三个注解，其中一个是@EnableAutoConfiguration注解，这个注解主要是开启自动配置的，这个注解会"猜"你将如何配置 spring，前提是你已经添加 了 jar 依赖项，比如项目中引入了 spring-boot-starter-web ，这个包里已经添加 Tomcat 和 SpringMVC，这个注解节就会自动假设您在开发一个 web 应用程序并添加相应的 spring 配置，springboot默认有一个spring-boot-autoconfigure包，大多数常用的第三方的配置都自动集成了，像redis、es等，这里边有一个`META-INF/spring.factories`文件，这里边定义了所有需要加载的bean的全路径，spring会根据反射的原理，创建这些对象，放到IOC容器中，加载时需要的参数，通过JavaConfig的方式加载配置文件中的参数然后创建了对应的对象，这就是自动配置的原理













