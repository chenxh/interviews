### Spring Boot 有哪些优点？
* 简化Spring初始搭建以及开发过程，在短时间内快速构建项目
* SpringBoot集成了大量的常用第三方库，例如Redis、Mongodb、JPA等，编码非常简单
* SpringBoot提供了丰富的starter，集成主流的开源产品，只需要少量配置或零配置即可
* SpringBoot内嵌了容器，通过简单命令就可以启动

### Spring boot 配置加载顺序

1、开发者工具 Devtools 全局配置参数；

2、单元测试上的 @TestPropertySource 注解指定的参数；

3、单元测试上的 @SpringBootTest 注解指定的参数；

4、命令行指定的参数，如 java -jar springboot.jar --name="Java程序鱼"；

5、命令行中的 SPRING_APPLICATION_JSONJSON 指定参数, 如 java -Dspring.application.json='{"name":"Java技术栈"}' -jar springboot.jar

6、ServletConfig 初始化参数；

7、ServletContext 初始化参数；

8、JNDI参数（如 java:comp/env/spring.application.json）；

9、Java系统参数（来源：System.getProperties()）；

10、操作系统环境变量参数；

11、RandomValuePropertySource 随机数，仅匹配：ramdom.*；

12、JAR包外面的配置文件参数（application-{profile}.properties（YAML））

13、JAR包里面的配置文件参数（application-{profile}.properties（YAML））

14、JAR包外面的配置文件参数（application.properties（YAML））

15、JAR包里面的配置文件参数（application.properties（YAML））

16、@Configuration配置文件上 @PropertySource 注解加载的参数；

17、默认参数（通过 SpringApplication.setDefaultProperties 指定）；


### Spring Boot 中的监视器是什么？
Spring boot actuator 是 spring 启动框架中的重要功能之一。Spring boot 监视器可帮助您访问生产环境中正在运行的应用程序的当前状态。有几个指标必须在生产环境中进行检查和监控。即使一些外部应用程序可能正在使用这些服务来向相关人员触发警报消息。监视器模块公开了一组可直接作为 HTTP URL 访问的REST 端点来检查状态。

### Spring Boot 打成的 jar 和普通的 jar 有什么区别 ?
Spring Boot 项目最终打包成的 jar 是可执行 jar ，这种 jar 可以直接通过 java -jar xxx.jar 命令来运行，这种 jar 不可以作为普通的 jar 被其他项目依赖，即使依赖了也无法使用其中的类。

Spring Boot 的 jar 无法被其他项目依赖，主要还是他和普通 jar 的结构不同。普通的 jar 包，解压后直接就是包名，包里就是我们的代码，而 Spring Boot 打包成的可执行 jar 解压后，在 \BOOT-INF\classes 目录下才是我们的代码，因此无法被直接引用。如果非要引用，可以在 pom.xml 文件中增加配置，将 Spring Boot 项目打包成两个 jar ，一个可执行，一个可引用。


### 如何自定义一个 spring boot starter？
https://blog.csdn.net/TaloyerG/article/details/132479543


1. 创建一个普通项目，引入 spring-boot-starter 依赖。
2. 创建属性类，例如： TestProperties.java。 注解 ：@ConfigurationProperties(prefix="test")。 此类用来指定获取对应配置
3. 创建自定配置类。 例如：TestAutoConfiguration.java。 注解 ：@Configuration。 此类用来指定自动配置。
```
@Configuration
@EnableConfigurationProperties(TestProperties.class)
@ConditionalOnProperty(prefix = "test", name = "enabled", havingValue = "true")

```
4. 在 resources 目录下创建 META-INF/spring.factories 文件。让Spring Boot能够在启动时自动扫描并加载我们的starter。 内容如下：
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.TestAutoConfiguration
```


### Starter项目的原理？
Starter项目的核心原理是基于Spring Boot的自动装配机制，即根据类路径和配置文件中的条件，动态地创建和注入Bean对象到应用上下文中。Starter项目通过在resources/META-INF目录下创建一个名为spring.factories的文件，来指定自动配置类的全限定名，让Spring Boot在启动时能够扫描并加载它。自动配置类则通过一系列的注解来定义和控制自动装配的逻辑，例如：

@Configuration注解标识这是一个配置类，用来创建和注册Bean对象。
@EnableConfigurationProperties注解启用属性类，并将其注入到配置类中。
@ConditionalOnClass注解判断类路径中是否存在指定的类，如果存在则满足条件。
@ConditionalOnProperty注解判断配置文件中是否存在指定的属性，如果存在并且值与期望相符，则满足条件。
@Bean注解根据属性类和业务功能类，创建相应类型的Bean对象，并注册到应用上下文中。
当所有的条件都满足时，Starter项目就能实现自动装配的功能，让我们无需手动编写和配置Bean对象，只需引入依赖和设置属性即可。


