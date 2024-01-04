## nacos 
https://github.com/alibaba/nacos
打包： mvn clean install  -Prelease-nacos


## nacos 启动。
startup.sh 启动 nacos-server.jar. 
mainClass 为 com.alibaba.nacos.Nacos

## 注册中心的原理

### nacos 与 spring-cloud 的集成
在spring-cloud-commons包中有一个类org.springframework.cloud. client.serviceregistry.ServiceRegistry ,它是Spring Cloud提供的服务注册的标准。集成到Spring Cloud中实现服务注册的组件,都会实现该接口。
nacos 实现了此接口。 NacosServiceRegistry。

***NacosServiceRegistry 的调用过程***
在spring-cloud-commons包的META-INF/spring.factories中包含自动装配的配置信息如下：
```
# AutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.client.CommonsClientAutoConfiguration,\
org.springframework.cloud.client.ReactiveCommonsClientAutoConfiguration,\
org.springframework.cloud.client.discovery.composite.CompositeDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.composite.reactive.ReactiveCompositeDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.simple.SimpleDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.simple.reactive.SimpleReactiveDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.hypermedia.CloudHypermediaAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.AsyncLoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.LoadBalancerDefaultMappingsProviderAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.LoadBalancerBeanPostProcessorAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.ReactorLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.client.serviceregistry.ServiceRegistryAutoConfiguration,\
org.springframework.cloud.commons.httpclient.HttpClientConfiguration,\
org.springframework.cloud.commons.util.UtilAutoConfiguration,\
org.springframework.cloud.configuration.CompatibilityVerifierAutoConfiguration,\
org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration,\
org.springframework.cloud.commons.security.ResourceServerTokenRelayAutoConfiguration,\
org.springframework.cloud.commons.config.CommonsConfigAutoConfiguration
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.cloud.client.HostInfoEnvironmentPostProcessor
# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.cloud.configuration.CompatibilityNotMetFailureAnalyzer
```

AutoServiceRegistrationAutoConfiguration 就是服务注册相关的配置类, 注入了 AutoServiceRegistration。 
AutoServiceRegistration 是一个接口， NacosAutoServiceRegistration 是 nacos的 实现类。
NacosAutoServiceRegistration  继承了 AbstractAutoServiceRegistration ， AbstractAutoServiceRegistration 实现了AutoServiceRegistration， ApplicationListener<WebServerInitializedEvent> 
当 WebServerInitializedEvent 事件到达时， AbstractAutoServiceRegistration 先设置端口， 然后启动q注册。
```
public void start() {
        if (!isEnabled()) {
            if (logger.isDebugEnabled()) {
                logger.debug("Discovery Lifecycle disabled. Not starting");
            }
            return;
        }
        // only initialize if nonSecurePort is greater than 0 and it isn't already running
        // because of containerPortInitializer below
        if (!this.running.get()) {
            this.context.publishEvent(new InstancePreRegisteredEvent(this, getRegistration()));
            register();
            if (shouldRegisterManagement()) {
                registerManagement();
            }
            this.context.publishEvent(new InstanceRegisteredEvent<>(this, getConfiguration()));
            this.running.compareAndSet(false, true);
        }
    }
```
最终会调用NacosServiceREgistry.register()方法进行服务注册。

***NacosServiceRegistry的实现***





