## SpringBoot 启动过程

SpringBoot 的启动类的 run() 方法开始看，以下是调用链：
    SpringApplication.run() -> 
        run(new Class[]{primarySource}, args) -> 
            new SpringApplication(primarySources)).run(args)。

一直在run，终于到重点了，我们直接看 new SpringApplication(primarySources)).run(args) 这个方法。

上面的方法主要包括两大步骤：

* 创建 SpringApplication 对象。
* 运行 run() 方法。

## SpringBoot 自动配置原理


### @SpringBootApplication 注解

@SpringBootApplication 标注在某个类上说明：

* 这个类是 SpringBoot 的主配置类。
* SpringBoot 就应该运行这个类的 main 方法来启动 SpringBoot 应用。

```
public @interface SpringBootApplication {
    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    Class<?>[] exclude() default {};

    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    String[] excludeName() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackages"
    )
    String[] scanBasePackages() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackageClasses"
    )
    Class<?>[] scanBasePackageClasses() default {};
}
```

* @SpringBootConfiguration：该注解表示这是一个 SpringBoot 的配置类，其实它就是一个 @Configuration 注解而已。
* @ComponentScan：开启组件扫描。
* @EnableAutoConfiguration：从名字就可以看出来，就是这个类开启自动配置的。嗯，自动配置的奥秘全都在这个注解里面。

### EnableAutoConfiguration 
```

@AutoConfigurationPackage 
@Import({AutoConfigurationImportSelector.class}) 
public @interface EnableAutoConfiguration {}
```

**@AutoConfigurationPackage**
@AutoConfigurationPackage 注解就是将主配置类（@SpringBootConfiguration标注的类）的所在包及下面所有子包里面的所有组件扫描到Spring容器中。
所以说，默认情况下主配置类包及子包以外的组件，Spring 容器是扫描不到的。

**@Import({AutoConfigurationImportSelector.class})**

我们知道 AutoConfigurationImportSelector 的 selectImports 就是用来返回需要导入的组件的全类名数组的，那么如何得到这些数组呢？

在 selectImports 方法中调用了一个 getAutoConfigurationEntry() 方法。

getAutoConfigurationEntry()方法的调用链： getAutoConfigurationEntry() -> getCandidateConfigurations() -> loadFactoryNames()。

loadFactoryNames() 中关键的三步：

* 从当前项目的类路径中获取所有 META-INF/spring.factories 这个文件下的信息。
* 将上面获取到的信息封装成一个 Map 返回。
* 从返回的 Map 中通过刚才传入的 EnableAutoConfiguration.class 参数，获取该 key 下的所有值。


AutoConfigurationImportSelector的作用是将类路径下 META-INF/spring.factories 里面配置的所有 EnableAutoConfiguration 的值加入到 Spring 容器中。

### HttpEncodingAutoConfiguration 示例说明
```
@Configuration 
@EnableConfigurationProperties({HttpProperties.class}) 
@ConditionalOnWebApplication( 
type = Type.SERVLET 
) 
@ConditionalOnClass({CharacterEncodingFilter.class}) 
@ConditionalOnProperty( 
prefix = "spring.http.encoding", 
value = {"enabled"}, 
matchIfMissing = true 
) 
public class HttpEncodingAutoConfiguration { 
```

@Configuration：标记为配置类。
@ConditionalOnWebApplication：web应用下才生效。
@ConditionalOnClass：指定的类（依赖）存在才生效。
@ConditionalOnProperty：主配置文件中存在指定的属性才生效。
@EnableConfigurationProperties({HttpProperties.class})：启动指定类的ConfigurationProperties功能；将配置文件中对应的值和 HttpProperties 绑定起来；并把 HttpProperties 加入到 IOC 容器中


